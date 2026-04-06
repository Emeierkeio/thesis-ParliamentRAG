---
created: 2026-04-06T21:08:29.945Z
title: Timeline feature with daily debates recap
area: ui
files:
  - frontend/src/app/page.tsx
  - backend/app/routers/query.py
  - backend/app/services/retrieval/engine.py
---

## Problem

The system currently only supports topic-based queries ("What is the position of groups on X?"). There is no way to browse parliamentary activity chronologically — users cannot see what was discussed on a specific day, across both Camera and Senato.

## Solution

New feature: **Parliamentary Timeline**

- A timeline view showing days with parliamentary sessions (both Camera and Senato)
- Each day shows the list of debates/discussions held that day
- Clicking on a debate shows a recap of who said what (speakers, key quotes, positions)
- Should leverage existing Session, Debate, Phase, Speech data in Neo4j
- Could reuse the DirectWriter for per-debate summaries

### UI concept
- Calendar/timeline component in a new page or sidebar tool
- Day cards showing session number, debate titles, chamber badge
- Debate detail: multi-view recap (similar to current topic summarization but scoped to a single debate)

### Data requirements
- Query: `MATCH (s:Session)-[:HAS_DEBATE]->(d:Debate) WHERE s.date = $date RETURN ...`
- Group speeches by speaker within each debate
- Support both chambers with chamber filter
