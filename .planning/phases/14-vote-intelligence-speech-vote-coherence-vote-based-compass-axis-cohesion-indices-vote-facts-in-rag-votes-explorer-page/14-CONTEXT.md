# Phase 14 Context: Vote intelligence

**Gathered:** 2026-07-05 — user approved all 5 proposals from the vote-data discussion ("voglio sviluppare tutte le tue proposte dalla 1 alla 5").
**Depends on:** Phase 8 complete (aggregate votes both chambers + individual votes: Camera resume to 391 deputies, Senate individuals). Features 1-3 are DEGRADED without individual votes — plan them to work at party level with aggregates and light up fully when individuals land.

<domain>
## Phase Boundary

Turn the vote graph (Vote, IndividualVote, VOTED, ON_VOTE, HAS_VOTE) into user-facing intelligence across five features. No new data ingestion — phase 8 owns that. Both legislatures, both chambers, legislature/chamber filters respected everywhere (phase 12 pattern; remember the coverage-fill lesson: EVERY new query joins Session and filters chamber+legislature).

</domain>

<decisions>
## Implementation Decisions

### F1 — Speech-vote coherence (the flagship)
- In chat responses: when citations reference a debate whose session has votes on the same act/subject, attach a "come hanno votato" block per cited party: outcome + party breakdown (from IndividualVote aggregated by MEMBER_OF_GROUP at vote date; fallback: no per-party block if individuals missing, show only overall outcome).
- Matching strategy vote↔debate: votes are Session-level; associate via the debate the citation came from (same session) + act linkage when present (Debate-DISCUSSES-Act); do NOT attempt NLP matching of vote subject in v1.
- UI: a compact editorial block under the citation group (hairline rule style, Fraunces numerals) — "Esito: approvato 143-92" + per-party favor/against/abstain chips when available.

### F2 — Vote-based compass axis
- New computation: party × vote matrix for the selected legislature+chamber (IndividualVote aggregated to party majority position per vote; abstain=0.5). Pairwise agreement → PCA/MDS to 1-2 dims. Pure Python (numpy already in backend), computed on demand with cache (votes are static per legislature).
- Endpoint: GET /api/compass/votes?legislature=&chamber= returning party coordinates + variance explained.
- UI: toggle on the compass page "Assi testuali | Assi di voto" (segmented, editorial style). Text compass stays default.
- Thesis note in UI tooltip: text axis = what they say, vote axis = what they do.

### F3 — Cohesion & rebellion indices
- Per party: Rice index per legislature (+ optional per-theme via debate-act linkage). Per parliamentarian: rebellion rate (voted != own party majority), participation rate (votes cast / votes held while in office).
- Surface on rankings page as new columns/toggle and on the speaker hover/detail. Needs individual votes — hide (not zero) when data absent for that chamber/legislature.
- Endpoint: GET /api/rankings/votes?legislature=&chamber=.

### F4 — Vote facts in RAG responses
- Zero-LLM enrichment: in the generation step, when retrieved citations belong to sessions with votes on a DISCUSSES-linked act, inject a structured fact line into the prompt context ("[FATTO] Il DDL X è stato approvato il {date} con {favor} sì, {against} no, {abstained} astenuti") and instruct the writer to use it verbatim when relevant.
- Verification: vote facts are graph lookups, exempt from citation-verifier (they carry their own vote id reference rendered as a chip linking to the debate).

### F5 — Votes explorer page
- New route /votes: filterable table (chamber, legislature, date range, esito, margine) with editorial favor/against bars, linked to the parent debate/session; click-through to DebateDetail.
- Follows the editorial design language (hairline rows, serif numerals, no cards). Sidebar entry under Strumenti.
- Data: extend existing /api/data/sessions/{id}/votes with a paginated cross-session /api/votes search endpoint.

### Ordering
1. Backend endpoints + indices computations (F2/F3 math, F5 search endpoint, F1/F4 lookups) — parallelizable
2. F4 (RAG injection) + F1 (chat UI block)
3. F5 page + F2 compass toggle + F3 rankings columns
4. E2E verify

### Claude's Discretion
- Exact PCA implementation (sklearn if present, else numpy SVD)
- Rice index edge cases (unanimous votes excluded per literature)
- Caching strategy for vote matrices (in-process TTL fine)

</decisions>

<canonical_refs>
## Canonical References

- `.planning/phases/08-*/08-RESEARCH.md` — vote node shapes, id conventions (camera_leg19_sedNNN_vNNN / senato_...), IndividualVote schema
- `backend/app/routers/data.py` — existing votes endpoint to extend
- `backend/app/services/timeline_service.py` get_debate_detail — vote→debate association precedent
- `backend/app/routers/query.py` + generation service — where F4 fact injection goes (same place expert/citation context is built)
- `frontend/src/app/compass/page.tsx`, `frontend/src/app/rankings/page.tsx` — pages to extend
- Phase 12 lesson (12-06-SUMMARY): every new Cypher joins Session with chamber+legislature filters — INCLUDING any post-merge augmentation
</canonical_refs>

<specifics>
## Data state at writing (2026-07-05)
- Aggregates: Camera XIX complete (16,506), Camera XVIII complete (11,558), Senato XIX 249/427 sittings (7,587), Senato XVIII 0 — resume pending dati.senato.it rate-ban lift
- Individuals: Camera XIX 69/391 deputies (471k), Senato 0 — run 08-07 pending
- Vote.subject from SPARQL is generic ("Votazione") — F5 display should prefer the DISCUSSES act title or debate title over subject
</specifics>

<deferred>
## Deferred Ideas
- NLP matching of vote subject text to specific amendments
- W-NOMINATE proper (ideal point MCMC) — PCA/MDS is enough for v1; cite as future work in thesis
- Cross-legislature compass comparison view
</deferred>

---

*Phase: 14-vote-intelligence*
*Context gathered: 2026-07-05 from product discussion*
