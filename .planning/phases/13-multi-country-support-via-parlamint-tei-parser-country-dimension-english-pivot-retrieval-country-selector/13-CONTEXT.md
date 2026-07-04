# Phase 13 Context: Multi-country support via ParlaMint

**Gathered:** 2026-07-04 — architectural decisions made by the system's lead developer session (user delegated: "scegli tu come").
**Depends on:** Phase 12 complete (country dimension stacks on top of the legislature dimension and reuses its exact propagation pattern).

<domain>
## Phase Boundary

Add foreign parliaments (pilot: **GB, FR, ES**) as a queryable dimension: chat with legislature-quality citations, search, compass. Italy remains the live, first-class citizen (own scrapers, votes, atti, authority). Foreign countries are comparative corpora with a reduced-but-honest feature set.

</domain>

<decisions>
## Implementation Decisions (LOCKED — rationale included so they survive context loss)

### D1 — Source: ParlaMint 5.0 (CLARIN.SI), NOT native parliament APIs
- One parser (Parla-CLARIN TEI/XML) covers 29 countries vs N bespoke scrapers.
- Uniform speaker/party/role metadata per speech — solves per-country biographical pipelines for free.
- `ParlaMint-en.ana` provides sentence-aligned English machine translation of every corpus → enables English-pivot retrieval (D4).
- Static ZIP downloads from clarin.si repository — no WAF, no scraping fragility.
- Tradeoff accepted: corpus releases lag reality by months. Foreign = comparative/historical; Italy = live via existing camera/senato pipelines. State this in the UI ("Dati aggiornati al" per country from corpus metadata).

### D2 — Pilot scope: GB, FR, ES — most recent full parliamentary term only
- 3 countries prove generality without 29× cost.
- Latest-term-only bounds embedding cost (~$1-3/country with text-embedding-3-small, shielded by the SQLite cache) and Neo4j size (~300-600k chunks/country).
- The parser/downloader MUST take country code + term as parameters so scaling to more countries is a config change, not code.

### D3 — Schema: extend, never fork
- `Session.country` (ISO 3166-1 alpha-2 lowercase: 'it', 'gb', 'fr', 'es'), default/migration: existing data → 'it'.
- Node id prefixes: `gb_term58_sed...` mirroring `sen_leg19_...` convention.
- Deputy nodes with `country`; ParliamentaryGroup per country (name collisions impossible across countries by id namespacing).
- Backend: `country: str = "it"` in QueryRequest/ChatRequest, propagated EXACTLY like `legislature` (Phase 12 pattern — same files, same clause style: `AND s.country = $country`). Zero behavior change until frontend sends otherwise.
- Coalition (maggioranza/opposizione): ParlaMint marks government/opposition status for most corpora; where absent, balance metrics degrade gracefully (flag `coalition_data: false` in response meta).

### D4 — Retrieval: English pivot for foreign countries
- Embeddings + BM25 computed on the **English MT text** (ParlaMint-en.ana), NOT on the original language. Cross-lingual retrieval with 3-small degrades; EN-pivot is the robust choice.
- Chunk stores BOTH: `text` = English (retrieval), `text_original` = source language (display/citation). Sentence alignment ids from ParlaMint make the mapping exact.
- Query path for country != 'it': translate query → EN via existing `_translate_text` service (parametrize target lang), retrieve on EN, generate citing `text_original` verbatim + EN rendering (reuse the citation translation UX from Phase 05 inverted).
- Full-text index: per-country index with english analyzer (`chunk_fulltext_en`) — the existing italian-analyzer index stays untouched.
- Italy keeps its current native-Italian path — NO regression risk.

### D5 — Feature parity is explicitly tiered (document in UI, not silently broken)
- Foreign countries: chat ✓, search ✓, compass ✓ (works on retrieved text), timeline ✗ (no AI summaries — cost), votes ✗, atti ✗.
- Authority v2 for foreign: reduced components (interventions count, institutional role from TEI, party role) — profession/education/acts/committees unavailable in ParlaMint. Authority response carries `components_available` list so the frontend can render honest tooltips.

### D6 — Frontend: country selector above legislature selector
- Same segmented-control pattern (editorial style, Phase 12 LegislatureSelector as template). Flag emoji + country name.
- Legislature selector becomes country-aware: 'it' → XVIII/XIX; foreign → the pilot term (single option, disabled state).
- Welcome badges + trending topics per country (extend TOPICS_BY_LEGISLATURE → keyed by country+term; curate 8-12 real topics per pilot country from the corpus).
- Rotating landing-page quotes stay Italian (landing is the product's identity).

### Claude's Discretion
- Chunker settings for EN text (probably same 1200/250), NER: skip spaCy for foreign (it_core_news_lg is Italian-only) — leave lawRefs/personRefs empty rather than loading 3 more models.
- Exact CLARIN.SI download URLs/handles per corpus version.
- Whether GB Commons+Lords or Commons only (recommend Commons only for the pilot).

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Existing patterns to replicate
- `.planning/phases/12-*/12-CONTEXT.md` and `12-RESEARCH.md` — the legislature dimension: same propagation, same ordering constraint (backend filter BEFORE ingest)
- `build/senate_parser.py` — the parsed-dict contract every parser must emit
- `build/db_builder.py` — ingest_session, ROMAN_MAP pattern, get_existing_session_numbers(chamber, legislature) → will need country param
- `backend/app/services/retrieval/{dense,sparse,ner,graph}_channel.py` — filter clause anchors
- `backend/app/services/translation.py` — `_translate_text` to parametrize for query→EN
- `frontend/src/components/chat/LegislatureSelector.tsx` + `use-chat.ts` — selector pattern

### External
- ParlaMint 5.0: https://www.clarin.si/repository/xmlui/handle/11356/1859 (and -en.ana variant)
- Parla-CLARIN TEI schema: https://github.com/clarin-eric/ParlaMint

</canonical_refs>

<specifics>
## Suggested plan split (6 plans, ordering constraint identical to Phase 12)

1. **13-01 Backend country filter** (FIRST — same poisoning risk as Phase 12): `country='it'` default in request models + retrieval channels + timeline; migration `SET s.country='it'` where null.
2. **13-02 ParlaMint downloader + TEI parser**: CLARIN.SI zips for GB/FR/ES latest term (both original and -en.ana), `build/parlamint_parser.py` emitting the senate_parser dict contract + `text_original`, speaker/party extraction from TEI headers.
3. **13-03 Ingest pipeline**: db_builder country param, EN fulltext index, chunker on EN with original alignment, speaker nodes from TEI metadata; ingest ONE country (GB) behind a human cost checkpoint, then FR/ES.
4. **13-04 Query path**: query→EN translation for country != it, generation prompt with original-language citations, citation verifier on text_original.
5. **13-05 Frontend**: CountrySelector, country-aware LegislatureSelector, badges/topics, tiered-feature messaging.
6. **13-06 E2E**: "What does the House of Commons think about immigration?" returns verbatim Commons citations; Italian XIX behavior byte-identical.

</specifics>

<deferred>
## Deferred Ideas
- Remaining 26 ParlaMint countries (config-only after pilot)
- Cross-country comparison view ("stesso tema, 4 parlamenti")
- Foreign timeline summaries (LLM cost), votes via national APIs (EP/Bundestag/Commons have them)
- EU Parliament via data.europarl.europa.eu (native API, multilingual verbatim — best candidate for country #4)
</deferred>

---

*Phase: 13-multi-country-support-via-parlamint*
*Context gathered: 2026-07-04 — decisions delegated to and made by the development session*
