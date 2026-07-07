---
phase: 14-vote-intelligence-speech-vote-coherence-vote-based-compass-axis-cohesion-indices-vote-facts-in-rag-votes-explorer-page
plan: "06"
subsystem: frontend-chat
tags: [vote-intelligence, sse, chat-ui, f1-vote-coherence, f4-vote-facts, typescript]
dependency_graph:
  requires: ["14-04", "14-05"]
  provides: ["vote_coherence SSE captured in use-chat", "VoteCoherenceBlock rendered under citations", "vote_facts chips rendered amber visually distinct"]
  affects: ["frontend/src/types/chat.ts", "frontend/src/hooks/use-chat.ts", "frontend/src/components/chat/MessageBubble.tsx"]
tech_stack:
  added: []
  patterns: ["SSE accumulator pattern (mirrors experts/compass)", "derived open state (avoids setState-in-useEffect)", "editorial hairline section with Fraunces numerals"]
key_files:
  created: []
  modified:
    - frontend/src/types/chat.ts
    - frontend/src/hooks/use-chat.ts
    - frontend/src/components/chat/MessageBubble.tsx
decisions:
  - "VoteCoherenceBlock rendered inside the citations CollapsibleSection (not as a peer section) to visually tie votes to the cited sessions"
  - "Vote-fact chips styled amber — distinct from blue citation chips and from the editorial VoteCoherenceBlock — exempt from citation-verification"
  - "citedSessionIds fallback: when no citation has session_id (e.g. empty set), all voteCoherence keys are shown rather than hiding the block entirely"
  - "[Rule 1 - Bug] Replaced setState-in-useEffect in CollapsibleSection with derived open state (isOpen || forceOpen) — eliminates react-hooks/set-state-in-effect ESLint error"
  - "[Rule 1 - Bug] Replaced unescaped quote characters in VariantCard with &ldquo;&rdquo; entities — fixes react/no-unescaped-entities ESLint error"
metrics:
  duration: "6min"
  completed: "2026-07-07"
  tasks_completed: 3
  files_modified: 3
---

# Phase 14 Plan 06: Vote Coherence + Vote-Fact Chips in Chat Summary

**One-liner:** F1 speech-vote coherence editorial block and F4 vote-fact amber chips wired end-to-end from SSE to MessageBubble with full TypeScript typing and clean ESLint output.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Add voteCoherence to Message type + capture SSE in use-chat | 880e66e | types/chat.ts, hooks/use-chat.ts |
| 2 | Render "come hanno votato" editorial block in MessageBubble | b5fc8ae | components/chat/MessageBubble.tsx |
| 3 | Capture vote_facts SSE + render F4 vote-fact chips | 3debc95 | components/chat/MessageBubble.tsx |

## What Was Built

### Types Added (`frontend/src/types/chat.ts`)
- `VoteCoherencePartyBreakdown` — party + favor/against/abstain counts
- `VoteCoherenceVote` — single vote record with party_breakdown
- `VoteCoherenceSession` — votes array + debate_id per session
- `VoteCoherenceData` — `Record<string, VoteCoherenceSession>` (keyed by session_id)
- `VoteFactChip` — vote_id + debate_id + label for F4 chips
- `session_id?: string` added to `Citation` (populated by citation_details SSE from query.py line 418)
- `voteCoherence?` and `voteFacts?` added to `Message` interface

### SSE Capture (`frontend/src/hooks/use-chat.ts`)
- Imports: `VoteCoherenceData`, `VoteFactChip`
- Accumulators: `let voteCoherence` and `let voteFacts` declared alongside experts/compass
- `case "vote_coherence":` → sets accumulator + calls `updateLastAssistantMessage`
- `case "vote_facts":` → sets accumulator + calls `updateLastAssistantMessage`
- Both fields included in `"complete"` assembly (main loop and buffered flush path)

### VoteCoherenceBlock (`frontend/src/components/chat/MessageBubble.tsx`)
- Rendered inside the Citations `CollapsibleSection` after citation cards
- Matches `message.voteCoherence` keys to `message.citations[].session_id` set
- Per vote: aggregate row (outcome + Fraunces `[font-family:var(--font-display)]` serif numerals)
- Per-party chips with green favor / red against / neutral abstain counts
- Falls back to `tvc('onlyAggregate')` when all breakdown counts are zero (Pitfall 3 guard)
- Hidden entirely when no citation session_id matches voteCoherence

### VoteFactChips (F4)
- Rendered below Compass section in `AssistantMetadata`
- Each chip is `<a href="/transcript/{debate_id}">` or `/timeline` fallback
- Amber styling (`bg-amber-50 border-amber-200`) — visually distinct from citation chips
- `aria-label={tvf('chipAria')}` for accessibility
- Not fed into any citation-verification path

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Fixed unescaped quote entities in VariantCard**
- **Found during:** Task 2 ESLint run
- **Issue:** `"{message.hqMetadata!.judge_reason}"` used raw `"` characters in JSX — `react/no-unescaped-entities` error
- **Fix:** Replaced with `&ldquo;...&rdquo;`
- **Files modified:** `frontend/src/components/chat/MessageBubble.tsx`
- **Commit:** b5fc8ae

**2. [Rule 1 - Bug] Eliminated setState-in-useEffect in CollapsibleSection**
- **Found during:** Task 2 ESLint run
- **Issue:** `useEffect(() => { if (forceOpen) setIsOpen(true); }, [forceOpen])` triggered `react-hooks/set-state-in-effect` error
- **Fix:** Derived open state as `const derivedOpen = isOpen || forceOpen;` — same behavior, no cascading renders
- **Files modified:** `frontend/src/components/chat/MessageBubble.tsx`
- **Commit:** b5fc8ae

## Verification Results

- `npx tsc --noEmit -p tsconfig.json` — exits 0 (no TypeScript errors)
- `npx eslint src/components/chat/MessageBubble.tsx src/hooks/use-chat.ts` — 0 errors, only pre-existing unused-var warnings

## Self-Check: PASSED
