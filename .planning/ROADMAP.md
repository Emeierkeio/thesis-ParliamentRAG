### Phase 14: Vote intelligence: speech-vote coherence, vote-based compass axis, cohesion indices, vote facts in RAG, votes explorer page

**Goal:** The existing vote graph (Vote, IndividualVote, VOTED, ON_VOTE, HAS_VOTE) becomes five user-facing intelligence features: F1 speech-vote coherence blocks under chat citations, F2 a vote-based compass axis toggle, F3 party cohesion + deputy rebellion/participation indices, F4 verified vote facts injected into RAG answers, and F5 a filterable votes explorer page. No new data ingestion — pure analytics + API + UI. Every new query filters chamber+legislature (Phase 12 rule); features needing IndividualVote data degrade gracefully (aggregate-only or hidden) while Camera XIX / Senate individual votes are still landing from Phase 8.
**Requirements**: VI-01, VI-02, VI-03, VI-04, VI-05
**Depends on:** Phase 13
**Plans:** 2/8 plans executed

Plans:
- [ ] 14-01-PLAN.md — votes_service.py: Rice/rebellion/participation math + vote facts/coherence/search lookups + Wave 0 tests (VI-01, VI-03, VI-04, VI-05)
- [ ] 14-02-PLAN.md — Vote compass PCA pipeline (numpy SVD) + GET /api/compass/votes (VI-02)
- [ ] 14-03-PLAN.md — votes router: GET /api/votes (F5 search) + GET /api/rankings/votes (F3) + main.py include (VI-03, VI-05)
- [ ] 14-04-PLAN.md — F4 vote facts injection into DirectWriter + F1 vote_coherence SSE event in query.py (VI-01, VI-04)
- [ ] 14-05-PLAN.md — Frontend types + votes-api client + /votes explorer page + sidebar entry + all Phase-14 i18n (VI-05)
- [ ] 14-06-PLAN.md — F1 chat block: vote_coherence SSE capture + MessageBubble "come hanno votato" render (VI-01)
- [ ] 14-07-PLAN.md — F2 compass vote/text toggle + F3 rankings cohesion columns (VI-02, VI-03)
- [ ] 14-08-PLAN.md — E2E verification: automated gates + human live-DB checkpoint (VI-01..VI-05)
