---
phase: 06-senate-data-integration-with-chamber-selector
verified: 2026-04-04T00:00:00Z
status: passed
score: 14/14 must-haves verified
re_verification: false
---

# Phase 6: Senate Data Integration with Chamber Selector — Verification Report

**Phase Goal:** The system ingests Senato della Repubblica stenographic records (XIX legislatura) alongside Camera data, with a chamber selector (Camera / Senato / Both) in the UI that filters retrieval queries.
**Verified:** 2026-04-04
**Status:** PASSED
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| #  | Truth                                                                       | Status     | Evidence                                                                                     |
|----|-----------------------------------------------------------------------------|------------|----------------------------------------------------------------------------------------------|
| 1  | senate_parser.py parses AKN XML, returns same dict structure as Camera      | VERIFIED   | `SenateStenograficoParser.parse_xml_file()` exists, all 6 unit tests pass                   |
| 2  | PRESIDENTE speeches excluded from Senate output                             | VERIFIED   | Dual filtering: `as` attr check + `an:from` text fallback in `_parse_speeches()`            |
| 3  | Session IDs use `sen_` prefix to avoid collision with Camera                | VERIFIED   | `session_id = f"sen_leg19_sed{number}"` at line 209                                         |
| 4  | download_senate.py fetches AKN files from senato.it with User-Agent header  | VERIFIED   | `headers = {"User-Agent": USER_AGENT}` at line 65; `download_senate_xmls()` defined         |
| 5  | make db-senate builds Senate data additively (no nuke)                      | VERIFIED   | `do_build_senate()` has no `nuke_database` call; comment says "additive — no nuke"          |
| 6  | make db-all calls Camera build then Senate build                            | VERIFIED   | Makefile db-all: Camera `build` at step 4, then `build-senate` before "Database ready!"    |
| 7  | ChatRequest accepts a chamber field with default "both"                     | VERIFIED   | `chamber: str = Field(default="both", ...)` at chat.py line 48                             |
| 8  | All 3 retrieval channels accept a chambers parameter                        | VERIFIED   | `chambers: list[str] | None = None` in dense, sparse, graph channel `retrieve()` signatures |
| 9  | Dense channel Cypher includes `WHERE s.chamber IN $chambers`                | VERIFIED   | Line 74 in dense_channel.py; `"chambers": chambers` in params dict at line 110             |
| 10 | Sparse channel Cypher includes `WHERE s.chamber IN $chambers`               | VERIFIED   | Line 77 in sparse_channel.py; `"chambers": chambers` in params dict                        |
| 11 | Graph channel Cypher includes `WHERE s.chamber IN $chambers` (both queries) | VERIFIED   | 4 occurrences: retrieve(), _get_chunks_from_signatories(), _get_chunks_by_entity()          |
| 12 | ChamberSelector renders 3 options: Camera, Senato, Both                     | VERIFIED   | `OPTIONS: ChamberValue[] = ["camera", "senato", "both"]` iterated in JSX render            |
| 13 | Selection persists in localStorage across page reloads                      | VERIFIED   | `localStorage.getItem("parliamentRAG.chamber")` in init; `setItem` in useEffect            |
| 14 | Chamber value sent to backend in chat request body                          | VERIFIED   | `body: JSON.stringify({ query: content, task_id: taskId, chamber })` at use-chat.ts line 166 |

**Score:** 14/14 truths verified

---

### Required Artifacts

| Artifact                                                       | Expected                                      | Status     | Details                                                                   |
|----------------------------------------------------------------|-----------------------------------------------|------------|---------------------------------------------------------------------------|
| `build/senate_parser.py`                                       | AKN XML parser, SenateStenograficoParser      | VERIFIED   | 330+ lines; class, parse_xml_file, PRESIDENTE filtering, sen_ prefix      |
| `build/download_senate.py`                                     | Senate XML downloader with User-Agent         | VERIFIED   | download_senate_xmls(), User-Agent header, rate limiting                  |
| `build/build_and_update.py`                                    | do_build_senate + build-senate CLI mode       | VERIFIED   | Imports both modules; do_build_senate() at line 356; CLI at line 412      |
| `build/tests/test_senate_parser.py`                            | 6 unit tests for Senate parser                | VERIFIED   | 6 tests collected, 6 passed in 0.02s                                      |
| `build/fixtures/sample_session.akn`                            | Minimal valid AKN fixture                     | VERIFIED   | File exists and drives all 6 passing tests                                |
| `Makefile`                                                     | db-senate target + db-all updated             | VERIFIED   | db-senate at line 337; build-senate called in db-all at line 284          |
| `backend/app/routers/chat.py`                                  | ChatRequest.chamber field                     | VERIFIED   | Field(default="both") at line 48; chambers computed and passed at 235/692 |
| `backend/app/services/retrieval/engine.py`                     | chambers parameter threaded to all 3 channels | VERIFIED   | chambers param in retrieve_sync and retrieve; passed to all 3 channels    |
| `backend/app/services/retrieval/dense_channel.py`              | WHERE s.chamber IN $chambers                  | VERIFIED   | Line 74 in Cypher; chambers in params                                     |
| `backend/app/services/retrieval/sparse_channel.py`             | WHERE s.chamber IN $chambers                  | VERIFIED   | Line 77 in Cypher; chambers in params                                     |
| `backend/app/services/retrieval/graph_channel.py`              | WHERE s.chamber IN $chambers (both queries)   | VERIFIED   | 4 grep hits covering both internal methods                                |
| `frontend/src/components/chat/ChamberSelector.tsx`             | Segmented button, 3 options, useTranslations  | VERIFIED   | export function ChamberSelector; OPTIONS array; useTranslations call      |
| `frontend/messages/it.json`                                    | ChamberSelector locale keys                   | VERIFIED   | "ChamberSelector": { camera, senato, both: "Entrambi", label }            |
| `frontend/messages/en.json`                                    | ChamberSelector locale keys                   | VERIFIED   | "ChamberSelector": { camera: "Chamber of Deputies", senato, both, label } |
| `frontend/src/hooks/use-chat.ts`                               | chamber state, localStorage, body inclusion   | VERIFIED   | useState init reads localStorage; useEffect persists; body includes field |
| `frontend/src/components/chat/ChatArea.tsx`                    | imports + renders ChamberSelector             | VERIFIED   | import at line 9; <ChamberSelector value={chamber} ... /> at line 88      |
| `frontend/src/app/chat/[id]/page.tsx`                          | destructures chamber/setChamber from useChat  | VERIFIED   | chamber, setChamber destructured; passed as props to ChatArea             |

Note: `frontend/src/app/chat/page.tsx` as named in the plan does not exist — the actual chat page is at `frontend/src/app/chat/[id]/page.tsx`. The wiring is fully implemented there; the path discrepancy is a plan documentation artifact, not a functional gap.

---

### Key Link Verification

| From                                      | To                                         | Via                                      | Status  | Details                                                   |
|-------------------------------------------|--------------------------------------------|------------------------------------------|---------|-----------------------------------------------------------|
| build/senate_parser.py                    | build/xml_parser.py                        | from xml_parser import StenograficoParser | WIRED   | Line 18: `from xml_parser import StenograficoParser, classify_phase_type` |
| build/build_and_update.py                 | build/senate_parser.py                     | from senate_parser import                | WIRED   | Line 49                                                   |
| build/build_and_update.py                 | build/download_senate.py                   | from download_senate import              | WIRED   | Line 50                                                   |
| backend/app/routers/chat.py               | backend/app/services/retrieval/engine.py   | passes chambers to retrieve_sync/retrieve | WIRED  | chambers computed at line 235, passed to retrieve_sync at 238; line 692/698 for streaming |
| backend/app/services/retrieval/engine.py  | dense_channel.py / sparse_channel.py / graph_channel.py | chambers=chambers in all 3 run_* calls | WIRED | Lines 134, 141, 151 in engine.py                     |
| frontend/src/components/chat/ChamberSelector.tsx | frontend/messages/it.json           | useTranslations("ChamberSelector")       | WIRED   | Line 16 in ChamberSelector.tsx; key exists in both locale files           |
| frontend/src/components/chat/ChatArea.tsx | frontend/src/components/chat/ChamberSelector.tsx | import + render ChamberSelector   | WIRED   | Import line 9; JSX at line 88                             |
| frontend/src/hooks/use-chat.ts            | backend (ChatRequest)                      | chamber in JSON.stringify body           | WIRED   | Line 166; backend ChatRequest.chamber default="both"      |

---

### Requirements Coverage

| Requirement | Source Plan | Description                                                                                 | Status    | Evidence                                                      |
|-------------|-------------|---------------------------------------------------------------------------------------------|-----------|---------------------------------------------------------------|
| SEN-01      | 06-01       | Dedicated Senate XML parser producing same output format as Camera parser                   | SATISFIED | SenateStenograficoParser passes all 6 unit tests; same dict keys as Camera parser |
| SEN-02      | 06-01       | Senate data download and ingestion into Neo4j with `chamber: "senato"` on all nodes        | SATISFIED | download_senate.py downloads AKN files; do_build_senate ingests via DatabaseBuilder.ingest_session (which sets chamber from parsed dict); session_id uses sen_ prefix |
| SEN-03      | 06-03       | Chamber selector UI (Camera/Senato/Both) above chat input, default "Both"                  | SATISFIED | ChamberSelector.tsx renders 3 segmented buttons; default is "both" from localStorage or useState init |
| SEN-04      | 06-02       | All retrieval channels filter by chamber when not "Both"                                    | SATISFIED | WHERE s.chamber IN $chambers in all 3 channels; chambers list computed from request.chamber string |
| SEN-05      | 06-01       | make db-senate for Senate-only build, make db-all builds both chambers                     | SATISFIED | db-senate target at Makefile line 337; db-all calls build then build-senate sequentially |

No orphaned requirements detected.

---

### Anti-Patterns Found

No blockers or warnings found. No TODO/FIXME/PLACEHOLDER comments in any modified files. No stub implementations. No empty handlers.

---

### Human Verification Required

#### 1. Senate data actually ingested in Neo4j

**Test:** Run `make db-senate` against a live Neo4j instance with at least one `.akn` file in `data/senate_xml/`. Then query `MATCH (s:Session {chamber: "senato"}) RETURN count(s)`.
**Expected:** Count > 0; sessions have `chamber = "senato"`.
**Why human:** Requires live Neo4j + network access to senato.it; cannot verify programmatically in this context.

#### 2. Chamber selector visual rendering and placement

**Test:** Open the chat UI in a browser. Verify the segmented button group (Camera / Senato / Entrambi) appears above the chat input area. Click each option and verify it highlights correctly.
**Expected:** 3 clearly labeled buttons; active option visually distinguished; layout does not break on mobile.
**Why human:** Visual/CSS behavior cannot be verified by static code analysis.

#### 3. Chamber filter produces different result sets

**Test:** With Senate data ingested, send the same query twice: once with chamber="camera", once with chamber="senato". Compare source citations in the response.
**Expected:** Camera query returns only Camera stenographic sources; Senato query returns only Senate sources; "Entrambi" returns mix from both.
**Why human:** Requires live data and a running backend to observe actual retrieval behavior.

---

### Gaps Summary

No gaps. All 14 observable truths verified, all 17 artifacts pass all three levels (exists, substantive, wired), all 5 key links confirmed, and all 5 requirements satisfied. The only noted discrepancy — plan 03 referencing `frontend/src/app/chat/page.tsx` when the actual file is `frontend/src/app/chat/[id]/page.tsx` — does not represent a functional gap; the wiring is fully implemented in the actual route file.

---

_Verified: 2026-04-04_
_Verifier: Claude (gsd-verifier)_
