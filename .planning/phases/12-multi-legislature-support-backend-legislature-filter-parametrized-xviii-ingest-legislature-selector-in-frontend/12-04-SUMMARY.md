---
phase: 12-multi-legislature-support-backend-legislature-filter-parametrized-xviii-ingest-legislature-selector-in-frontend
plan: 04
subsystem: data
tags: [neo4j, ingest, xviii, embeddings]

requires:
  - phase: 12
    plan: 01
    provides: legislature filter live in retrieval (poisoning guard)
  - phase: 12
    plan: 03
    provides: parametrized build pipeline + make db-ingest-leg18

provides:
  - XVIII legislature fully ingested: 741 Camera + 459 Senate sessions (legislature=18)
  - 660 XVIII deputies + 336 XVIII senators loaded with groups/committees
  - Speaker coverage: Camera 95.6%, Senato 92.2% (orphans = Conte I/II and Draghi government members — documented gap, app_config.py has XIX roles only)
  - Embeddings 100% coverage (0 chunks without embedding)

affects: [12-06 E2E verification, phase 13 multi-country]

tech:
  discovered:
    - Run interrupted by SIGTERM at 453/741; resume via legislature-aware get_existing_session_numbers worked exactly as designed (found 288 remaining)
---

# 12-04: XVIII additive ingest — COMPLETE

One-liner: XVIII (2018-2022) ingested additively next to XIX: 1,200 sessions, ~60k speeches, zero impact on XIX behavior (default-19 filters live since 12-01).

Human checkpoint: approved via "launch-now" (2026-07-03). Run log: /tmp/leg18-ingest.log + /tmp/leg18-ingest2.log (resume).

## Verification
- `MATCH (s:Session {legislature: 18})` → camera 741, senato 459 ✓ (gate > 1000 passed)
- Chunks without embedding: 0 ✓
- Speaker link coverage per legislature/chamber measured and recorded above.
