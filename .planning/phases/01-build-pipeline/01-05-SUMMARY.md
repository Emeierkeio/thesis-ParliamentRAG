---
plan: "01-05"
phase: "01-build-pipeline"
status: complete
started: "2026-04-02"
completed: "2026-04-02"
duration: "~20 min (including full build)"
---

# Plan 01-05: Integration Verification

## Result: PASS

## What Was Built

- `build/tests/test_integration.py` — Integration test suite with `@pytest.mark.integration` markers for schema verification against live Neo4j

## Self-Check: PASSED

### Verification Results (against live Neo4j after `make db-all`)

| Check | Expected | Actual | Status |
|-------|----------|--------|--------|
| Vote nodes | > 0 | 7,355 | PASS |
| DISCUSSES edges | > 0 | 2,144 | PASS |
| speakingRole on Speech | > 0 | 5,974 | PASS |
| Session-[:HAS_VOTE]->Vote | > 0 | 7,355 | PASS |
| Debate-[:HAS_VOTE]->Vote | 0 | 0 | PASS |
| Phase.phaseType populated | > 0 | 6,874 | PASS |
| Chunk.start_char_raw (removed) | 0 | 0 | PASS |
| Speech.preprocessed_text (removed) | 0 | 0 | PASS |
| Session.completeDate (removed) | absent | absent | PASS |
| Chunk keys | [text, index, embedding, id] | [text, index, embedding, id] | PASS |

### Database Statistics

| Node Label | Count |
|-----------|-------|
| Session | 638 |
| Debate | 6,312 |
| Phase | 6,874 |
| Speech | 41,671 |
| Chunk | 155,517 |
| Vote | 7,355 |
| Deputy | 391 |
| GovernmentMember | 64 |
| ParliamentaryAct | 1,530 |

### Phase Type Enum Values Found

`ballot`, `discussion`, `reply`, `vote_declaration`, `vote`, `resolution_announcement`, `government_opinion`, `general_discussion`, `other`, `article_examination`

## Deviations

None.

## key-files

### created
- `build/tests/test_integration.py`

### modified
- None (verification only)
