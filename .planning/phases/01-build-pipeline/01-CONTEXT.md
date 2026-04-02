# Phase 1: Build Pipeline - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Rebuild the database build pipeline from scratch with a clean English-only Neo4j schema, batch writes, managed transactions, and new data extraction (Vote nodes, Debate-to-Act links, Speaker roles, Phase types). The output is a single `make db-all` command that rebuilds the entire database with fresh data. The Italian-schema save path is removed; only the XML parser logic is retained (extracted to its own module).

</domain>

<decisions>
## Implementation Decisions

### Schema Design — Minimal properties per node

**Speech node:**
- Keep only `text` (preprocessed content) — remove raw text, `preprocessedText` duplicate, and `surnameNam`
- Speaker reconciliation must be complete at build time (no runtime fallback needed)
- Add `speakingRole` property for institutional role from `<emphasis>` XML tag

**Session node:**
- Remove `completeDate` string — use only Neo4j Date `date` property
- Keep: `id`, `legislature`, `number`, `year`, `month`, `day`, `chamber`, `date`

**Chunk node:**
- Minimal: `id`, `text`, `index`, `embedding`
- Remove: `charCount`, `startCharRaw`, `endCharRaw`
- Remove all alignment_map logic from chunking code

**All nodes:**
- camelCase property names (Neo4j Cypher style guide)
- PascalCase node labels (already correct)
- SCREAMING_SNAKE_CASE relationship types (already correct)

### Vote Extraction

- Node label: `Vote`
- Schema predisposed for individual vote records (Phase 4 SPARQL enrichment will add `IndividualVote` nodes)
- Properties from XML: `number`, `type`, `subject`, `present`, `voters`, `abstained`, `majority`, `inFavor`, `against`, `onMission`, `outcome`
- Relationship: `Debate-[:HAS_VOTE]->Vote`
- Parse logic already exists in `parse_votazione()` — wire it into `_save_session_english()`

### Debate→Act Linking

- Parse `<argomenti>` XML element for act references (`tipologiaAtto` + `codiceArgomento`)
- Relationship: `Debate-[:DISCUSSES]->ParliamentaryAct`
- Matching strategy: match by URI/number against existing ParliamentaryAct nodes; create placeholder node if act not found in DB
- 5 act types found across 365 XML files: `pdl` (3,493), `interrogazioneRispostaOrale` (1,016), `mozione` (351), `doc` (264), `interpellanza` (194)

### Build Script Architecture

- **Modular structure:** Split into separate files by responsibility:
  - `xml_parser.py` — XML parsing (extracted from `ingest_stenografici.py`, no Neo4j dependency)
  - `chunker.py` — Sentence splitting, chunk grouping, overlap logic (no alignment_map)
  - `db_builder.py` — Neo4j schema creation, UNWIND batch writes, managed transactions
  - `csv_loader.py` — Deputy/Group/Committee loading from CSV
  - `download.py` — XML and CSV download from Camera APIs
  - `build_and_update.py` — CLI entry point orchestrating the above modules
- **Configuration:** YAML/JSON config file for chunking parameters (CHUNK_SIZE, OVERLAP, MIN_LENGTH) — separate from code
- **Batch writes:** UNWIND with batch size 1000
- **Transactions:** `execute_read`/`execute_write` managed transactions (replace auto-commit `session.run()`)

### Phase Type Enum

- Parse Phase title patterns into `phaseType` property
- Vocabulary: `government_opinion`, `vote_declaration`, `general_discussion`, `question_time`, `personal_fact`, etc.
- Pattern matching on the Italian title string

### Claude's Discretion
- Exact module file layout within `build/` directory
- YAML config file name and structure
- How to handle XML files with malformed structure (error recovery)
- Exact UNWIND Cypher query structure
- Phase type enum values (derive from actual title patterns in XML corpus)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Build pipeline code
- `build/build_and_update.py` — Current monolithic build script (1078 lines) — the code being refactored
- `build/ingest_stenografici.py` — XML parser + Italian save path (parser to extract, save to delete)
- `build/ingest_atti_parlamentari.py` — Parliamentary acts ingestion (referenced by builder)
- `build/precalculate_embeddings.py` — Embedding pre-calculation (must not break)
- `build/embedding_service.py` — Embedding service with cache key "text-embedding-3-small" (FROZEN)

### Research findings
- `.planning/research/STACK.md` — Neo4j naming conventions, batch write patterns, transaction management
- `.planning/research/FEATURES.md` — XML data extraction opportunities (Votes, argomenti, speaker roles, phase types)
- `.planning/research/ARCHITECTURE.md` — Current vs target architecture, dependency graph
- `.planning/research/PITFALLS.md` — Critical risks (parser coupling, embedding cache, schema rename blast radius)

### Neo4j schema
- `.planning/research/ENRICHMENT.md` — Vote schema design, future individual vote structure

### XML data source
- `data/xml/stenografico_leg19_*.xml` — Raw XML files to analyze for extraction

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `StenograficoIngester.parse_xml_file()` — Full XML parsing (sessions, debates, phases, speeches, votes) — extract to `xml_parser.py`
- `StenograficoIngester.preprocess_text_with_alignment()` — Text preprocessing (parenthesis removal, speaker markers) — keep logic, remove alignment_map
- `StenograficoIngester.create_chunks()` — Sentence-aware chunking — extract to `chunker.py`, simplify
- `StenograficoIngester.merge_continuation_interventions()` — Ellipsis merge logic — keep in parser
- `StenograficoIngester.parse_votazione()` — Vote parsing (already complete!) — wire into save path
- `DatabaseBuilder` class — Neo4j write operations — refactor with UNWIND batch writes

### Established Patterns
- `build/app_config.py` — Role definitions (GOVERNMENT_ROLES, PARLIAMENT_ROLES, etc.) — keep as-is
- CSV loading with pandas + `clean_generic_label()`, `extract_group_info()` — keep utility functions
- `download_deputies_csv.py` — SPARQL download from dati.camera.it — keep as separate download module

### Integration Points
- `Makefile` — `db-populate`, `db-update`, `db-all` targets call `build_and_update.py`
- `precalculate_embeddings.py` — Runs after build, reads Chunk/Speech/Committee/Act nodes
- `precalculate_baseline_experts.py` — Requires backend running, called separately
- `embedding_service.py` + `neo4j_helper.py` — Shared by embeddings script and build

</code_context>

<specifics>
## Specific Ideas

- The user wants "expert-level" code quality — clean, explainable, best-practice Python
- Type hints everywhere, English docstrings, no dead code
- The parser extraction from `ingest_stenografici.py` is the critical first step (STATE.md decision)
- Embedding cache key string "text-embedding-3-small" must not change (STATE.md blocker)

</specifics>

<deferred>
## Deferred Ideas

- Individual vote records per deputy (SPARQL from dati.camera.it) — Phase 4 ENR-01
- NER entity extraction on chunks — Phase 4 ENR-03/04
- BM25 sparse retrieval channel — Phase 4 RET-01

</deferred>

---

*Phase: 01-build-pipeline*
*Context gathered: 2026-04-02*
