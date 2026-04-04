# Phase 6: Senate Data Integration - Research

**Researched:** 2026-04-04
**Domain:** Italian Senate XML (Akoma Ntoso AKN), build pipeline extension, Neo4j schema, FastAPI, Next.js
**Confidence:** HIGH (all critical findings verified by direct inspection and live data)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **Data source:** senato.it open data portal (stenographic XML records)
- **Legislature:** XIX only (current, consistent with Camera)
- **Parser:** Dedicated `build/senate_parser.py` module — separate from `xml_parser.py` (Camera)
- **Output format:** Same dict structure as Camera parser (`parse_xml_file()` returns same keys: `session`, `debates`, `phases`, `speeches`, `votes`, `act_references`)
- **db_builder.py receives uniform data** — no Camera/Senato distinction at the write level, only the `chamber` property differs
- **Makefile target:** `make db-senate` for Senate-only build, `make db-all` builds both
- **Component:** Dropdown/toggle above the chat input area
- **Options:** Camera | Senato | Entrambi (Both)
- **Default:** Entrambi (Both) — search across both chambers
- **Behavior:** Selection persists per session (localStorage). Filters the retrieval query.
- **i18n:** Labels translated in both Italian and English (Phase 5 infrastructure)
- **Backend propagation:** Chamber selection sent to backend via query parameter or request body field
- **Deputy distinction:** Same `Deputy` label with `chamber: "camera" | "senato"` property
- **Session distinction:** Same `Session` label with `chamber: "camera" | "senato"` property
- **GovernmentMember:** No `chamber` property — government members are cross-chamber
- **ParliamentaryGroup:** Senato has different group names than Camera — new groups created as needed
- **Committee:** Senato committees are different from Camera — new nodes created
- **Retrieval filter:** Dense and graph channels add `WHERE s.chamber IN $chambers` clause when not "both"

### Claude's Discretion
- Senate XML format analysis and parser implementation details
- Senate download URL patterns and error handling
- How to handle Senate-specific metadata differences
- Exact chamber selector component design (toggle vs dropdown)
- Whether Senate embeddings need separate pre-calculation or can share the existing pipeline
- How to handle cross-chamber government members in retrieval results

### Deferred Ideas (OUT OF SCOPE)
- Historical legislatures (XVIII and earlier) — future milestone
- Cross-chamber analysis features (comparing Camera vs Senato positions) — future milestone
- Joint session (seduta comune) handling — rare, defer
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SEN-01 | Dedicated Senate XML parser (`senate_parser.py`) producing same output format as Camera parser | AKN XML structure fully analyzed; output dict contract documented below |
| SEN-02 | Senate data download and ingestion into Neo4j with `chamber: "senato"` on all nodes | Download URL pattern confirmed; SPARQL session discovery verified; db_builder chamber field already exists |
| SEN-03 | Chamber selector UI component (Camera / Senato / Both) above chat input, default "Both" | ChatRequest model identified; use-chat.ts fetch call documented; locale key structure confirmed |
| SEN-04 | All retrieval channels (dense, sparse, graph) filter by `chamber` when not "Both" | All 3 channel Cypher queries analyzed; exact injection point identified |
| SEN-05 | `make db-senate` target for Senate-only build, `make db-all` builds both chambers | Makefile structure analyzed; build_and_update.py entry point documented |
</phase_requirements>

---

## Summary

Phase 6 adds the Italian Senate (Senato della Repubblica) as a second data source alongside Camera dei Deputati. The Senate publishes stenographic records in **Akoma Ntoso 3.0 XML** (`.akn` format), which is structurally very different from the Camera's custom XML. A dedicated `senate_parser.py` must translate the AKN format into the same output dict that `db_builder.py` already consumes.

The download discovery strategy uses a two-stage approach: the **dati.senato.it SPARQL endpoint** returns all 405 sessions (XIX legislature, Oct 2022–present) with session numbers and dates. Individual AKN files are then fetched from a confirmed URL pattern on senato.it. The db_builder already writes `chamber` on Session nodes (defaults to `"camera"`); Senate sessions simply pass `"senato"`. All three retrieval channels need a single `WHERE s.chamber IN $chambers` clause added to their Cypher queries.

On the frontend, a `ChamberSelector` component is added above the chat input. The `chamber` field is added to `ChatRequest` (backend Pydantic model) and the `body: JSON.stringify(...)` call in `use-chat.ts`. Locale keys follow the existing `it.json`/`en.json` pattern established in Phase 5.

**Primary recommendation:** Implement SEN-02 (download + parser) first, then SEN-04 (retrieval filter), then SEN-03 (UI). SEN-01 is a prerequisite for SEN-02 and SEN-05.

---

## Senate XML Structure (Verified — HIGH Confidence)

### Download URL Pattern

```
https://www.senato.it/leg/19/BGT/Testi/Resaula/{ID:08d}.akn
```

Example verified: `https://www.senato.it/leg/19/BGT/Testi/Resaula/01410431.akn` → Session 163, 2024-02-27.

The ID is a BGT (document management) numeric ID, **not** a sequential session number. These IDs are not predictably sequential — they are sparse across a large range.

### Session Discovery via SPARQL (Verified — HIGH Confidence)

The `dati.senato.it` SPARQL endpoint returns all assembly sessions for XIX legislature:

```sparql
-- Endpoint: https://dati.senato.it/sparql
SELECT ?s ?num ?date WHERE {
  ?s a <http://dati.senato.it/osr/SedutaAssemblea> ;
     <http://dati.senato.it/osr/numeroSeduta> ?num ;
     <http://dati.senato.it/osr/dataSeduta> ?date ;
     <http://dati.senato.it/osr/legislatura> ?leg .
  FILTER(?leg = 19)
} ORDER BY ?date
```

**Result:** Returns 405 sessions. Session 1 = 2022-10-13. Internal URIs are `http://dati.senato.it/sedutaassemblea/23908` etc.

**CRITICAL GAP:** The SPARQL does NOT provide the BGT document ID needed to construct the download URL. The BGT ID comes from the listing page HTML on senato.it (`show-doc?leg=19&tipodoc=Resaula&id={BGT_ID}&idoggetto=0`).

### Session Listing Page (Verified — HIGH Confidence)

```
https://www.senato.it/lavori/assemblea/resoconti-elenco-cronologico
```

This page only renders the **29 most recent** sessions. Filters (`?anno=`, `?mese=`) do NOT work — the page always returns the same 29 sessions. The listing HTML contains show-doc links:

```html
href="/show-doc?leg=19&tipodoc=Resaula&id=1499067&idoggetto=0"
```

### Recommended Download Strategy

`download_senate.py` should implement:

1. Scrape the listing page to extract the 29 most recent BGT IDs.
2. For each ID not already on disk, fetch `{ID:08d}.akn`.
3. Maintain a local `data/senate_xml/discovered_ids.json` file with `{session_num: bgt_id}` mappings.
4. Bootstrap historical sessions separately via a one-time `--bootstrap` mode that queries SPARQL for all 405 (num, date) pairs, then fetches each show-doc page to extract the BGT ID.

**IMPORTANT:** Always set `User-Agent: Mozilla/5.0 (...)` — the server returns 403/empty for requests without a browser UA.

**Verified known ID range:** Session 20 (2022-12-20) = ID 1363838. Session 405 (2026-04-01) = ID 1499067.

### AKN XML Root Structure

The Senate AKN XML uses the **Akoma Ntoso 3.0** namespace (`http://docs.oasis-open.org/legaldocml/ns/akn/3.0/CSD03`), abbreviated `an:` below.

```
an:akomaNtoso
  an:debate
    an:meta
      an:identification (FRBRWork: date, number, subtype=RESAULA)
      an:references (TLCPerson entries: person ID → last name)
    an:coverPage (group abbreviations block)
    an:debateBody
      an:debateSection* (name="InizioSeduta" | "Presidenza" | debate topic | "FineSeduta")
        an:heading?
        an:speech* (by="#p{personId}" as="#roleId")
          an:from (refersTo="#p{personId}", text=speaker display name)
          an:p* (text paragraphs)
        an:narrative?
        an:block?
        an:p?
    an:attachments
```

### Session Metadata Extraction

From `an:debate/an:meta/an:identification/an:FRBRWork`:

| AKN element | Attribute | Value example | Maps to |
|-------------|-----------|---------------|---------|
| `an:FRBRnumber` | `value` | `"163"` | session.number |
| `an:FRBRdate` | `date` | `"2024-02-27"` | session.date |
| `an:FRBRsubtype` | `value` | `"RESAULA"` | confirms it's an assembly session |

Constructing `session_id`: `f"sen_leg19_sed{number}"` (prefix `sen_` avoids collision with Camera `leg19_sed{num}` IDs).

Year/month/day are parsed from the `FRBRdate` date string.

### Speaker (Person) References

From `an:debate/an:meta/an:references/an:TLCPerson`:

```xml
<an:TLCPerson id="p32600"
              href="http://dati.senato.it/osr/Persona/32600"
              showAs="CASTELLONE"/>
```

- `id` attribute: `p{numeric_id}` — the numeric part (e.g., `32600`) is the senator's persistent ID on dati.senato.it.
- `showAs`: last name only (uppercase).
- `href`: `http://dati.senato.it/osr/Persona/{id}` — links to senator profile.

Speaker ID for the graph: `f"sen_{numeric_id}"` (prefix `sen_` avoids collision with Camera deputy IDs which are plain integers).

### Speech (debateSection/speech) Structure

```xml
<an:debateSection name="DiscussioneTopic">
  <an:heading>Discussion title</an:heading>
  <an:speech by="#p32600" as="#d1e27a1074">
    <an:from refersTo="#p32600">CASTELLONE</an:from>
    <an:p>Signor Presidente, ...</an:p>
    <an:p>Additional paragraph text.</an:p>
  </an:speech>
</an:debateSection>
```

- `speech.by`: `#p{numeric_id}` — speaker reference.
- `speech.as`: role reference (e.g., `#d1e27a1074` = PRESIDENTE). Filter PRESIDENTE speeches.
- `from.text`: speaker display name (last name, uppercase). Used as `cognome_nome`.
- `p` elements: collect all itertext() from all `an:p` children for the speech text.

**No ellipsis continuation merging needed** — Senate XML has complete speeches per `speech` element (no `interventoVirtuale` pattern).

**No `debattito`/`fase` nesting** — Senate structure uses flat `debateSection` elements. Map: each `debateSection` → one Debate node. No Phase nodes needed (or create one Phase per section with phaseType="other").

### Votes

Senate AKN XML does **not** embed vote data inline in session stenographic records. Votes are in separate documents. The `votes: []` key returns empty list — same as Camera sessions with no raccoltaVotazioni.

### Act References

Senate AKN does not have `<argomenti>` equivalents inline in the stenographic records. Return `act_references: {}` empty dict.

---

## Camera Parser Output Contract (parse_xml_file return dict)

This is what `senate_parser.py` MUST produce exactly:

```python
{
    "session": {
        "id": str,          # e.g. "sen_leg19_sed163"
        "legislature": int, # 19
        "number": int,      # session number from FRBRnumber
        "year": int,
        "month": int,
        "day": int,
        "chamber": str,     # "senato" (ALWAYS hardcoded for senate_parser)
        "date": str,        # "YYYY-MM-DD"
    },
    "debates": [            # one per debateSection with speeches
        {
            "id": str,          # f"{session_id}_{section_name_slug}"
            "originalId": str,  # debateSection name attribute
            "title": str,       # heading text
            "order": int,
            "sessionId": str,
        },
        ...
    ],
    "phases": [             # empty list OR one Phase per debate (no sub-phases in AKN)
        {
            "id": str,
            "originalId": str,
            "title": str,
            "phaseType": str,   # from classify_phase_type(heading)
            "order": int,
            "debateId": str,
        },
        ...
    ],
    "speeches": [
        {
            "id": str,          # f"{session_id}_sp{speech_index}"
            "originalId": str,
            "text": str,        # preprocessed (parentheticals removed)
            "testo_raw": str,   # raw concatenated paragraph text
            "deputatoId": str,  # f"sen_{numeric_person_id}" or None
            "cognome_nome": str, # from from.text (last name uppercase)
            "speakingRole": str | None, # e.g. "Presidente" if as= role
            "sessionId": str,
            "debateId": str,
            "phaseId": str,     # same as debateId-phase if creating one phase per debate
            "parentType": str,  # "phase" or "debate"
            "parentId": str,
            "order": int,
        },
        ...
    ],
    "votes": [],            # always empty for Senate AKN
    "act_references": {},   # always empty for Senate AKN
}
```

**Reuse `classify_phase_type()` and `preprocess_text()`** — import from `xml_parser.py` directly. The preprocessing logic (parenthetical removal, whitespace normalization) is language-agnostic and works for Senate text identically.

---

## Schema Impact Analysis (Verified — HIGH Confidence)

### db_builder.py — Session Chamber Field

Already handles chamber correctly in `_create_session()`:

```python
# Line 394, 403 in db_builder.py
SET s.chamber = $chamber
...
chamber=session_data.get("chamber", "camera"),
```

**No change needed to db_builder.py** for Session nodes. If `senate_parser.py` always puts `"chamber": "senato"` in the session dict, it flows through automatically.

### Deputy Nodes — Chamber Property

Current `_create_deputies()` in db_builder.py does NOT set a `chamber` property on Deputy nodes. Senate senators must get `chamber: "senato"`. Camera deputies currently have no `chamber` property — they should get `chamber: "camera"` retroactively.

**Schema change needed:**
1. `db_builder.py`: add `chamber` parameter to deputy creation Cypher.
2. For Camera deputies (existing): add a migration step or set `chamber: "camera"` in the deputy CSV loader.

Alternatively (simpler per the CONTEXT.md decision): Chamber is inferred from Session, not Deputy. When retrieving, speeches are linked to sessions which have `s.chamber`. The `Deputy.chamber` property is optional for filtering (the session-based filter alone may suffice).

**Recommendation:** Set `chamber` on Deputy nodes only from the senator CSV data. Camera deputies already ingested — add `chamber: "camera"` in `_create_deputies` with default fallback. Senators add `chamber: "senato"` explicitly.

### Retrieval Channel Cypher Modifications (Verified — HIGH Confidence)

All three channels traverse `(s:Session)` via the path:
```
Speech ← CONTAINS_SPEECH ← Phase ← HAS_PHASE ← Debate ← HAS_DEBATE ← Session
```

The `WHERE s.chamber IN $chambers` clause must be added **after** the Session MATCH.

#### DenseChannel (dense_channel.py, line 71-96)

```python
# Current:
MATCH (i)<-[:CONTAINS_SPEECH]-(f:Phase)<-[:HAS_PHASE]-(d:Debate)<-[:HAS_DEBATE]-(s:Session)

# After (add one line):
MATCH (i)<-[:CONTAINS_SPEECH]-(f:Phase)<-[:HAS_PHASE]-(d:Debate)<-[:HAS_DEBATE]-(s:Session)
WHERE s.chamber IN $chambers
```

Pass `chambers=["camera"]` / `["senato"]` / `["camera","senato"]`.

When `chambers = ["camera", "senato"]` (Both), pass both values → no filtering penalty.

#### SparseChannel (sparse_channel.py, line 73-74)

Same pattern — add after the Session MATCH.

#### GraphChannel (graph_channel.py)

Two Cypher queries need the filter:
- `_get_chunks_from_signatories()` (line 374-399): add after Session MATCH.
- `_get_chunks_by_entity()` (line 318-339): add after Session MATCH.

#### Neo4jClient vector_search() (neo4j_client.py, line 152-174)

This method is used by other parts of the system (not the retrieval channels which have their own Cypher). Leave unchanged — it doesn't participate in RAG retrieval pipelines directly.

---

## Camera Parser → Senate Parser Diff Summary

| Aspect | Camera (`xml_parser.py`) | Senate (`senate_parser.py`) |
|--------|--------------------------|------------------------------|
| Root element | `<seduta>` | `<an:akomaNtoso><an:debate>` |
| Namespace | None / xhtml only | `http://docs.oasis-open.org/legaldocml/ns/akn/3.0/CSD03` |
| Session metadata | Root attributes: `legislatura`, `numero`, `anno`, `mese`, `giorno`, `ramo` | FRBRWork: `an:FRBRnumber`, `an:FRBRdate` |
| Debate nodes | `<resoconto><dibattito id="...">` | `<an:debateBody><an:debateSection name="...">` |
| Phase nodes | `<fase id="...">` inside `dibattito` | None (flat structure — create one Phase per Section) |
| Speech nodes | `<intervento id="..."><testoXHTML>` | `<an:speech by="#p{id}"><an:p>...` |
| Speaker ID | `<nominativo id="...">` attribute | `speech.by` attribute = `#p{id}` |
| Speaker name | `nominativo.cognomeNome` attribute | `from.text` (uppercase last name) |
| PRESIDENTE filter | `nominativo.text == "PRESIDENTE"` | `speech.as` matches PRESIDENTE role ID |
| Votes | `<raccoltaVotazioni>` | None (return empty list) |
| Act refs | `<metadati><argomenti>` | None (return empty dict) |
| Ellipsis merging | Yes (ellipsis continuation) | No (complete speeches per element) |
| Chamber value | `root.get('ramo')` → "camera" | Always `"senato"` (hardcoded) |

---

## Architecture Patterns

### Build Pipeline Extension

```
build/
├── xml_parser.py           # Camera parser (unchanged)
├── senate_parser.py        # NEW — AKN parser, same interface
├── download.py             # Camera download (unchanged)
├── download_senate.py      # NEW — Senate download via listing scrape
├── db_builder.py           # Minor: add chamber to Deputy write
├── build_and_update.py     # Add do_build_senate() function
└── config.yaml             # Unchanged
```

### senate_parser.py Class Interface

```python
class SenateStenograficoParser:
    """Pure AKN parser for Senato della Repubblica stenografici.

    No Neo4j dependency. All output is plain Python dicts.
    Output format identical to StenograficoParser.parse_xml_file().
    """
    AKN_NS = 'http://docs.oasis-open.org/legaldocml/ns/akn/3.0/CSD03'

    def __init__(self, config: Optional[BuildConfig] = None) -> None: ...

    def parse_xml_file(self, filepath: str) -> dict:
        """Same return contract as StenograficoParser.parse_xml_file()."""
        ...
```

Import shared utilities from `xml_parser`:
```python
from xml_parser import classify_phase_type, StenograficoParser
# Reuse: StenograficoParser.preprocess_text() as a static-ish helper
```

### download_senate.py Interface

```python
SENATE_LISTING_URL = "https://www.senato.it/lavori/assemblea/resoconti-elenco-cronologico"
SENATE_AKN_URL = "https://www.senato.it/leg/19/BGT/Testi/Resaula/{:08d}.akn"
SENATE_SPARQL_URL = "https://dati.senato.it/sparql"
USER_AGENT = "Mozilla/5.0 (compatible; ParliamentRAG/1.0)"

def get_session_ids_from_listing(xml_dir: str) -> list[int]:
    """Fetch listing page and return BGT IDs not yet on disk."""

def download_senate_xmls(xml_dir: str) -> int:
    """Download new Senate AKN files. Returns count downloaded."""
```

**File naming convention:** `resaula_leg19_{session_num:04d}.akn` — uses session number (extracted from the downloaded file's FRBRnumber) for consistent naming on disk.

### build_and_update.py Extension

```python
def do_build_senate(uri, user, password, skip_download=False, skip_embeddings=False):
    """Build Senate portion of the database (do NOT nuke — additive)."""
    # 1. Download new Senate AKN files
    # 2. Parse each with SenateStenograficoParser
    # 3. Pass parsed dict to DatabaseBuilder.ingest_session() with chamber="senato"
    # 4. Run precalculate_embeddings for Senate chunks
```

Note: `do_build_senate()` is **additive** — does NOT call `nuke_database()`. This allows Camera and Senate data to coexist.

### Makefile db-senate Target

```makefile
db-senate: db-install ## Build Senate-only DB (additive, Camera data preserved)
    @printf "$(BOLD)$(CYAN)Senate database build...$(RESET)\n"
    @docker compose up -d neo4j
    # ... wait for neo4j ...
    @$(PYTHON) $(BUILD_SCRIPT) build-senate \
        --neo4j-uri $(NEO4J_LOCAL) \
        --neo4j-user $(NEO4J_USER) \
        --neo4j-password $(NEO4J_PASS)
```

### ChatRequest Extension (backend)

In `backend/app/routers/chat.py`, `ChatRequest` currently has:
```python
class ChatRequest(BaseModel):
    query: str = Field(...)
    mode: str = Field(default="standard")
    task_id: Optional[str] = Field(default=None)
    locale: str = Field(default="it")
```

Add:
```python
chamber: str = Field(default="both", description="Filter: 'camera' | 'senato' | 'both'")
```

Then pass `chambers` parameter to all three retrieval channel calls.

### use-chat.ts Extension

Current fetch call (line 149-156):
```typescript
body: JSON.stringify({ query: content, task_id: taskId }),
```

Add `chamber` to the hook options and the body:
```typescript
// In UseChatOptions or as a useState in the hook:
body: JSON.stringify({ query: content, task_id: taskId, chamber: selectedChamber }),
```

The `chamber` state can live in `ChatArea` or in `useChat` as a prop.

### ChamberSelector Component

Recommended: a segmented button group (3 states), not a dropdown. Placed between the ChatInput bar and the top sticky header area.

```tsx
// frontend/src/components/chat/ChamberSelector.tsx
export function ChamberSelector({ value, onChange }: {
  value: "camera" | "senato" | "both";
  onChange: (v: "camera" | "senato" | "both") => void;
}) { ... }
```

Persist to `localStorage` key `"parliamentRAG.chamber"` with default `"both"`.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| AKN XML parsing | Custom regex scraper | Python's `xml.etree.ElementTree` with namespace prefix | AKN is well-formed XML; ET handles namespaces via `{ns}tag` syntax |
| Senate session discovery | Custom web crawler | SPARQL query to `dati.senato.it/sparql` | Returns all 405 sessions reliably with num+date |
| Multi-value IN filter | ORM abstraction | Direct Cypher `WHERE s.chamber IN $chambers` | Neo4j handles list params natively |
| Chamber state persistence | Session storage or cookie | `localStorage` | Already used in project (NEXT_LOCALE uses cookie; chamber is UI-state only) |
| AKN namespace handling | String splitting on `}` | Define `NS = {'an': '...'}` dict and use `findall('an:tag', NS)` | Cleaner, ET-idiomatic |

---

## Common Pitfalls

### Pitfall 1: AKN Namespace in ElementTree
**What goes wrong:** `root.find('debate')` returns `None`. All AKN elements are namespaced.
**Why it happens:** ElementTree requires `{namespace}tag` or a `{'prefix': 'namespace'}` dict.
**How to avoid:**
```python
NS = {'an': 'http://docs.oasis-open.org/legaldocml/ns/akn/3.0/CSD03'}
debate = root.find('an:debate', NS)  # Correct
debate = root.find('debate')          # Wrong — returns None
```

### Pitfall 2: Missing User-Agent causes HTTP 403
**What goes wrong:** `curl` / `requests.get(url)` returns an HTML 403 page instead of XML.
**Why it happens:** senato.it rate-limits or blocks bare `python-requests` UA.
**How to avoid:** Always set `headers={"User-Agent": "Mozilla/5.0 (compatible; ParliamentRAG/1.0)"}` in the download script.

### Pitfall 3: PRESIDENTE speeches not filtered
**What goes wrong:** Parser includes presidency interventions (procedural announcements).
**Why it happens:** In AKN, the presiding senator speaks many times per session with short procedural text.
**How to avoid:** Check `speech.get('as', '')` against the TLCRole entries with `showAs="PRESIDENTE"`. Build a role-id-to-name lookup from `an:references/an:TLCRole`.

```python
# Build presidente role id set from TLCRole references
presidente_roles = {
    elem.get('id')
    for elem in refs.findall('an:TLCRole', NS)
    if elem.get('showAs', '').upper() == 'PRESIDENTE'
}
# Then in speech loop:
as_attr = speech.get('as', '').lstrip('#')
if as_attr in presidente_roles:
    continue  # skip
```

### Pitfall 4: Session ID collision with Camera
**What goes wrong:** Camera session 1 (`leg19_sed1`) and Senate session 1 (`leg19_sed1`) get the same Neo4j node ID.
**Why it happens:** Both use session number 1 for the first session of XIX legislature.
**How to avoid:** Prefix Senate session IDs with `sen_`: `f"sen_leg19_sed{number}"`. Camera IDs remain `f"leg{leg}_sed{num}"`.

### Pitfall 5: do_build_senate() must NOT call nuke_database()
**What goes wrong:** Running `make db-senate` wipes Camera data.
**Why it happens:** `do_build()` starts with `builder.nuke_database()`.
**How to avoid:** `do_build_senate()` is a separate function that skips the nuke. The Makefile `db-all` target calls `do_build()` (Camera) first, then `do_build_senate()` additively.

### Pitfall 6: `chambers` parameter not passed to all three retrieval channels
**What goes wrong:** Chamber filter applies in dense but not sparse or graph channels. Mixed results confuse users.
**Why it happens:** Each channel has independent Cypher with separate WHERE clauses.
**How to avoid:** Pass `chambers` from `ChatRequest` down to all three `.retrieve()` calls in `chat.py` and `query.py`. Add `chambers: list[str] = ["camera", "senato"]` parameter to each channel's `retrieve()` signature.

### Pitfall 7: BM25 full-text index covers both chambers
**What goes wrong:** Sparse channel ignores chamber filter when `chunk_fulltext` index is queried.
**Why it happens:** The full-text index indexes ALL Chunk nodes regardless of chamber. The WHERE clause on Session must be added after the MATCH chain.
**How to avoid:** Add `WHERE s.chamber IN $chambers` in `sparse_channel.py` after the Session MATCH (same as dense and graph channels).

---

## Code Examples

### AKN XML Parsing — Session Metadata

```python
# Source: verified against live file 01410431.akn (session 163, 2024-02-27)
import xml.etree.ElementTree as ET

NS = {'an': 'http://docs.oasis-open.org/legaldocml/ns/akn/3.0/CSD03'}

tree = ET.parse(filepath)
root = tree.getroot()
debate = root.find('an:debate', NS)
meta = debate.find('an:meta', NS)
ident = meta.find('an:identification', NS)
frbrwork = ident.find('an:FRBRWork', NS)

number = int(frbrwork.find('an:FRBRnumber', NS).get('value'))  # 163
date_str = frbrwork.find('an:FRBRdate', NS).get('date')         # "2024-02-27"
year, month, day = map(int, date_str.split('-'))
session_id = f"sen_leg19_sed{number}"
```

### AKN XML Parsing — Speaker References

```python
# Build person ID → last name lookup from meta.references
refs = meta.find('an:references', NS)
person_lookup: dict[str, str] = {}  # "p32600" → "CASTELLONE"
for tlc_person in refs.findall('an:TLCPerson', NS):
    person_lookup[tlc_person.get('id')] = tlc_person.get('showAs', '')

# Build PRESIDENTE role IDs set
presidente_roles: set[str] = {
    elem.get('id')
    for elem in refs.findall('an:TLCRole', NS)
    if elem.get('showAs', '').upper() == 'PRESIDENTE'
}
```

### AKN XML Parsing — Speech Loop

```python
debate_body = debate.find('an:debateBody', NS)
speech_global_idx = 0
debate_order = 0

for section in debate_body.findall('an:debateSection', NS):
    section_name = section.get('name', '')
    if section_name in ('InizioSeduta', 'FineSeduta', 'Presidenza'):
        continue  # Skip non-debate sections

    heading_elem = section.find('an:heading', NS)
    heading = ''.join(heading_elem.itertext()).strip() if heading_elem is not None else section_name

    debate_id = f"{session_id}_dib{debate_order}"
    debates.append({
        'id': debate_id,
        'originalId': section_name,
        'title': heading,
        'order': debate_order,
        'sessionId': session_id,
    })
    # One Phase per debate section
    phase_id = f"{debate_id}_fase0"
    phases.append({
        'id': phase_id,
        'originalId': f"{section_name}_fase0",
        'title': heading,
        'phaseType': classify_phase_type(heading),
        'order': 0,
        'debateId': debate_id,
    })

    speech_order = 0
    for speech_elem in section.findall('an:speech', NS):
        by_attr = speech_elem.get('by', '').lstrip('#')      # "p32600"
        as_attr = speech_elem.get('as', '').lstrip('#')       # "d1e27a1074"

        # Skip PRESIDENTE speeches
        if as_attr in presidente_roles:
            continue

        # Extract speaker info
        numeric_id = by_attr.lstrip('p') if by_attr.startswith('p') else None
        speaker_id = f"sen_{numeric_id}" if numeric_id else None
        cognome_nome = person_lookup.get(by_attr, '')

        # Collect text from all <an:p> elements
        paras = speech_elem.findall('an:p', NS)
        raw_text = ' '.join(''.join(p.itertext()) for p in paras).strip()

        # Reuse Camera preprocess_text
        from xml_parser import StenograficoParser
        _parser = StenograficoParser()
        clean_text = _parser.preprocess_text(raw_text)

        if len(clean_text) < config.min_speech_length:
            continue

        speeches.append({
            'id': f"{session_id}_sp{speech_global_idx}",
            'originalId': f"sp{speech_global_idx}",
            'text': clean_text,
            'testo_raw': raw_text,
            'deputatoId': speaker_id,
            'cognome_nome': cognome_nome,
            'speakingRole': None,  # Senate AKN doesn't use <emphasis> role tags
            'sessionId': session_id,
            'debateId': debate_id,
            'phaseId': phase_id,
            'parentType': 'phase',
            'parentId': phase_id,
            'order': speech_order,
        })
        speech_order += 1
        speech_global_idx += 1

    debate_order += 1
```

### Retrieval Channel Chamber Filter

```python
# Source: pattern to add in dense_channel.py, sparse_channel.py, graph_channel.py
# Add chambers param to retrieve() signature:
def retrieve(self, query_embedding, top_k=None, similarity_threshold=None,
             chambers: list[str] | None = None) -> list[dict]:
    chambers = chambers or ["camera", "senato"]  # default = both

# Add to Cypher (after Session MATCH):
cypher = """
...
MATCH (i)<-[:CONTAINS_SPEECH]-(f:Phase)<-[:HAS_PHASE]-(d:Debate)<-[:HAS_DEBATE]-(s:Session)
WHERE s.chamber IN $chambers
...
"""
results = self.client.query(cypher, {
    ...,
    "chambers": chambers,
})
```

### Locale Keys (i18n)

Add to `frontend/messages/it.json` and `en.json`:

```json
// it.json
"ChamberSelector": {
  "camera": "Camera",
  "senato": "Senato",
  "both": "Entrambi",
  "label": "Ramo del Parlamento"
}

// en.json
"ChamberSelector": {
  "camera": "Chamber",
  "senato": "Senate",
  "both": "Both",
  "label": "Parliamentary Chamber"
}
```

### ChatArea — ChamberSelector Placement

In `frontend/src/components/chat/ChatArea.tsx`, the selector goes inside the sticky top bar, above the ChatInput:

```tsx
// After line 60 (top sticky div), before <ChatInput>:
<ChamberSelector value={chamber} onChange={setChamber} />
<ChatInput
  onSend={(msg) => onSendMessage(msg, chamber)}
  ...
/>
```

The `chamber` state is lifted to `ChatArea` or managed in the parent page.

---

## Senate Group Names (Verified from Sample File)

From session 163 coverPage, the XIX legislature Senate groups are:

- `Cd'I-NM (UDC-CI-NcI-IaC)-MAIE` — Civici d'Italia-Noi Moderati
- `FI-BP-PPE` — Forza Italia-Berlusconi Presidente-PPE
- `FdI` — Fratelli d'Italia
- `IV-C-RE` — Italia Viva-Il Centro-Renew Europe
- `LSP-PSd'Az` — Lega Salvini Premier-Partito Sardo d'Azione
- `M5S` — MoVimento 5 Stelle
- `PD-IDP` — Partito Democratico-Italia Democratica e Progressista
- `AVS` — Alleanza Verdi e Sinistra
- `De.Sin.-FR` — De.Sin.-Fronte Repubblicano
- `Misto` — Misto

These differ from Camera group names. Each will be created as new `ParliamentaryGroup` nodes with their Senate names. The coalition logic (`backend/app/config.py`) needs to cover Senate group names for majority/opposition classification.

**Senate groups in XIX legislature (FdI-led majority):**
- **Maggioranza:** FdI, FI-BP-PPE, LSP-PSd'Az, Cd'I-NM (+ MAIE), Noi Moderati
- **Opposizione:** PD-IDP, M5S, IV-C-RE, AVS, De.Sin.
- **Governo:** government members (cross-chamber, no chamber property)

---

## State of the Art

| Old Approach | Current Approach | Impact |
|--------------|------------------|--------|
| Camera XML custom format | Senate uses AKN 3.0 standard | Requires different parser but standard format |
| Sequential session IDs (Camera: 0001...) | Senate uses BGT document IDs (non-sequential) | Download strategy must scrape listing page + store ID mapping |
| No chamber filtering | `s.chamber IN $chambers` WHERE clause | Single-chamber queries possible |
| `chatTitle` hardcoded to "Camera dei Deputati" | Should update to reflect selected chamber | MessageBubble.chamberTitle locale key |

---

## Open Questions

1. **Senator biographical data (CSV equivalent)**
   - What we know: Camera uses `deputati_xix.csv` from dati.camera.it for deputy names, groups, profession, etc.
   - What's unclear: There's no equivalent CSV download from dati.senato.it with senator biographical metadata in the same format.
   - Recommendation: Use `dati.senato.it/sparql` SPARQL to fetch senator names, groups, and biographical data. The SPARQL endpoint has `foaf:Person` entries for senators. This is a discovery task for the implementer.

2. **Senate coalition config (config.yaml/config.py)**
   - What we know: Current `backend/app/config.py` maps Camera group names to majority/opposition.
   - What's unclear: Senate group names are different — need to extend the coalition map.
   - Recommendation: Add Senate group names to `config.py` coalition mapping. Senate majority = same parties (FdI, FI, Lega, etc.) but different group name strings.

3. **Senator profile URLs**
   - What we know: Camera deputies have `camera_profile_url` links to their camera.it profile pages.
   - What's unclear: Senate senators have equivalent pages at `senato.it/Senatori/...` but the URL pattern is not confirmed.
   - Recommendation: Use `http://dati.senato.it/osr/Persona/{id}` as the canonical profile URL for senators (it redirects to their senato.it page). Low priority for MVP.

4. **`db-all` atomicity with Senate additive build**
   - What we know: `do_build()` nukes the database. `do_build_senate()` is additive.
   - What's unclear: Should `db-all` nuke first, then build Camera, then add Senate? Or nuke+build Camera+Senate together?
   - Recommendation: `db-all` = nuke (once) → `do_build()` Camera → `do_build_senate()` Senate. The nuke happens only once in `do_build()`. `do_build_senate()` never nukes.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest (existing, from Phase 2) |
| Config file | `backend/pytest.ini` or `backend/setup.cfg` (existing) |
| Quick run command | `pytest build/test_senate_parser.py -x` |
| Full suite command | `cd backend && python -m pytest -v --tb=short` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SEN-01 | `senate_parser.py` parses AKN XML and returns correct session/debates/phases/speeches | unit | `pytest build/test_senate_parser.py::test_parse_sample -x` | ❌ Wave 0 |
| SEN-01 | PRESIDENTE speeches are excluded from output | unit | `pytest build/test_senate_parser.py::test_presidente_filtered -x` | ❌ Wave 0 |
| SEN-01 | Session chamber is always "senato" | unit | `pytest build/test_senate_parser.py::test_chamber_value -x` | ❌ Wave 0 |
| SEN-02 | Session nodes in Neo4j have `chamber: "senato"` | smoke | `pytest backend/tests/test_senate_smoke.py -x` | ❌ Wave 0 |
| SEN-04 | Dense channel with `chambers=["camera"]` returns no Senate results | unit | `pytest backend/tests/test_retrieval_chamber.py::test_dense_camera_only -x` | ❌ Wave 0 |
| SEN-04 | Dense channel with `chambers=["camera","senato"]` returns both | unit | `pytest backend/tests/test_retrieval_chamber.py::test_dense_both -x` | ❌ Wave 0 |
| SEN-03 | ChamberSelector defaults to "both" on first mount | unit (jest/vitest) | `cd frontend && npx vitest run src/components/chat/ChamberSelector.test.tsx` | ❌ Wave 0 |
| SEN-03 | Chamber selection persists in localStorage | unit (jest/vitest) | `cd frontend && npx vitest run src/components/chat/ChamberSelector.test.tsx` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `pytest build/test_senate_parser.py -x -q` (< 5s, no Neo4j needed)
- **Per wave merge:** `cd backend && python -m pytest -v --tb=short`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `build/test_senate_parser.py` — covers SEN-01; requires sample AKN file fixture at `build/fixtures/sample_session.akn`
- [ ] `backend/tests/test_retrieval_chamber.py` — covers SEN-04; requires mock Neo4j or live DB
- [ ] `frontend/src/components/chat/ChamberSelector.test.tsx` — covers SEN-03; requires vitest
- [ ] Sample AKN fixture: copy `01410431.akn` to `build/fixtures/sample_session.akn` for unit tests

*(The existing Camera parser tests in `build/` provide the pattern for senate_parser tests.)*

---

## Sources

### Primary (HIGH confidence)
- Live Senate AKN XML file `01410431.akn` — analyzed Python-side; all XML elements verified
- `dati.senato.it/sparql` — SPARQL endpoint verified working; SedutaAssemblea query returns 405 sessions
- `build/xml_parser.py` — Camera parser source; output dict contract extracted directly
- `build/db_builder.py` — Session chamber field at lines 394, 403 verified
- `backend/app/services/retrieval/dense_channel.py` — Cypher query at lines 66-98 verified
- `backend/app/services/retrieval/sparse_channel.py` — Cypher query at lines 68-95 verified
- `backend/app/services/retrieval/graph_channel.py` — Cypher queries at lines 374-399 and 318-339 verified
- `frontend/src/hooks/use-chat.ts` — fetch body at line 155 verified
- `backend/app/routers/chat.py` — ChatRequest model at lines 42-48 verified

### Secondary (MEDIUM confidence)
- senato.it HTML source for listing page — shows 29 most recent BGT IDs; URL pattern confirmed
- `frontend/messages/it.json` and `en.json` — locale key structure verified

### Tertiary (LOW confidence)
- Senate group names — extracted from one session's coverPage; may have minor variations across sessions
- Senator CSV/biographical data source — SPARQL is confirmed available but specific query for senator metadata not tested

---

## Metadata

**Confidence breakdown:**
- Senate XML structure: HIGH — parsed live file, element paths verified in Python
- Download URL pattern: HIGH — confirmed working for IDs 1363838, 1410431, 1499067
- SPARQL session discovery: HIGH — query returns 405 results with correct session numbers and dates
- BGT ID discovery for historical sessions: MEDIUM — listing page only shows 29 most recent; historical bootstrap requires individual show-doc page scraping
- Schema impact: HIGH — all modified files read and analyzed
- Frontend integration: HIGH — ChatRequest, use-chat.ts, and ChatArea.tsx all verified

**Research date:** 2026-04-04
**Valid until:** 2026-05-04 (stable APIs; senato.it URL structure unlikely to change)
