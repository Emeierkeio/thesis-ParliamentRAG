---
phase: 14-vote-intelligence-speech-vote-coherence-vote-based-compass-axis-cohesion-indices-vote-facts-in-rag-votes-explorer-page
plan: "04"
subsystem: backend-query-pipeline
tags: [vote-intelligence, sse, direct-writer, rag-injection, f1, f4]
dependency_graph:
  requires: [14-01]
  provides: [vote_coherence-sse-event, vote_facts-sse-event, vote-facts-rag-injection]
  affects: [frontend-chat-14-06, direct_writer-prompt]
tech_stack:
  added: []
  patterns:
    - vote_facts injected as [FATTO DI VOTO] lines into DirectWriter LLM prompt
    - Non-fatal try/except guarding all three new lookups/emissions
    - _session_ids computed once from evidence_dicts; reused by both SSE events
    - vote_facts variable scoped in generator so Tasks 2 and 3 can access it
key_files:
  modified:
    - backend/app/services/generation/direct_writer.py
    - backend/app/routers/query.py
    - backend/tests/unit/test_sse_contract.py
  created: []
decisions:
  - "[14-04]: vote_facts and _session_ids initialised before gen_mode branch — both SSE events (F1, F4) degrade gracefully to no-op when pipeline mode is not 'direct' or data is absent"
  - "[14-04]: verified_citations do not carry session_id — vote_coherence falls back to _session_ids from evidence_dicts (same source as F4 injection)"
  - "[14-04]: TestChatEventTypes::test_chat_event_types_exist failure is pre-existing (chat.py missing 'waiting' event), unrelated to this plan — documented in deferred-items"
metrics:
  duration: "3 min"
  completed_date: "2026-07-07"
  tasks_completed: 3
  files_modified: 3
---

# Phase 14 Plan 04: Vote Facts RAG Injection + vote_coherence / vote_facts SSE Events Summary

Vote facts injected verbatim into the DirectWriter LLM prompt via [FATTO DI VOTO] lines, and two new SSE events (vote_coherence, vote_facts) emitted after citation_details for the chat UI.

## Tasks Completed

| # | Task | Commit | Key changes |
|---|------|--------|-------------|
| 1 | F4 — inject vote facts into DirectWriter prompt | 5c4552c | direct_writer.py vote_facts param + [FATTO DI VOTO] block; query.py vote_facts computation |
| 2 | F1 — emit vote_coherence SSE event after citation_details | 73d6aae | query.py vote_coherence emission; test_vote_coherence_event |
| 3 | F4 — emit vote_facts SSE event for frontend chip rendering | bd384e1 | query.py vote_facts chip emission; test_vote_facts_event |

## What Was Built

### Task 1: F4 — Vote facts injected into DirectWriter prompt

`direct_writer.py` now accepts `vote_facts: Optional[List[Dict[str, Any]]] = None` in both `generate()` and `_build_user_prompt()`. When non-empty, `_build_user_prompt` appends a `## FATTI DI VOTO` section with one `[FATTO DI VOTO]` line per vote fact (label, date, sì/no/astenuti, esito, vote_id). Both `_system_prompt_it` and `_system_prompt_en` now include a rule (rule 9/8) instructing the LLM to use these lines VERBATIM and not paraphrase vote numbers.

In `query.py`, immediately before `writer.generate()` in the `direct` mode branch:
- `_session_ids` is computed from `evidence_dicts` (set comprehension for uniqueness).
- `votes_service.get_vote_facts(services["neo4j"], _session_ids)` is called via `run_in_executor` (non-blocking).
- On failure, `vote_facts = []` is kept; error is logged as `[VOTE-FACTS] Failed (pipeline continues)`.
- `vote_facts=vote_facts` is passed to `writer.generate()`.

Both `vote_facts` and `_session_ids` are initialised as empty before the `gen_mode` conditional so they remain in scope for Tasks 2 and 3.

### Task 2: F1 — vote_coherence SSE event

After the experts-patch try/except block (~L501) a new guarded block calls `votes_service.get_vote_coherence(services["neo4j"], _coh_session_ids, request.legislature)`. Session IDs are taken from `verified_citations` (if they carry `session_id`) with fallback to `_session_ids`. The event is only emitted when `vote_coherence` is non-empty. Failure is logged as `[VOTE-COHERENCE] Failed (pipeline continues)`.

Contract test `TestVoteCoherenceAndFactsEvents::test_vote_coherence_event` source-inspects `query.py` to assert:
- `'type': 'vote_coherence'` is present.
- `votes_service.get_vote_coherence` is present.
- The line index of `vote_coherence` is strictly greater than the line index of `citation_details`.

### Task 3: F4 — vote_facts SSE event

Immediately after the vote_coherence block, a guarded block builds `_fact_chips = [{vote_id, debate_id, label}]` from the pre-computed `vote_facts` list and emits `{'type': 'vote_facts', 'data': _fact_chips}`. No DB re-query — reuses the `vote_facts` variable from Task 1. Failure is logged as `[VOTE-FACT-CHIPS] Failed (pipeline continues)`.

Contract test `TestVoteCoherenceAndFactsEvents::test_vote_facts_event` asserts presence, `_fact_chips` key, error guard string, and emission order relative to `citation_details`.

## Deviations from Plan

None — plan executed exactly as written. One pre-existing unrelated test failure was identified and documented below.

## Deferred Issues

**Pre-existing test failure (out of scope — not caused by this plan):**
`TestChatEventTypes::test_chat_event_types_exist` fails because `chat.py` does not emit a `'type': 'waiting'` event. This failure existed before plan 14-04 began (confirmed via stash verification). Logged for resolution in a future plan.

## Self-Check: PASSED

- `backend/app/services/generation/direct_writer.py` exists and parses: FOUND
- `backend/app/routers/query.py` exists and parses: FOUND
- `backend/tests/unit/test_sse_contract.py` exists: FOUND
- Commit 5c4552c: FOUND (Task 1)
- Commit 73d6aae: FOUND (Task 2)
- Commit bd384e1: FOUND (Task 3)
- `TestVoteCoherenceAndFactsEvents::test_vote_coherence_event`: 1 passed
- `TestVoteCoherenceAndFactsEvents::test_vote_facts_event`: 1 passed
