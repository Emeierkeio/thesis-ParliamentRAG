---
phase: 03-frontend
verified: 2026-04-02T19:00:00Z
status: gaps_found
score: 6/7 must-haves verified
re_verification: false
gaps:
  - truth: "Every SSE event payload has a typed interface derived from SSE_CONTRACT.md — and use-chat.ts imports SSE types for event parsing"
    status: partial
    reason: "SSEEvent discriminated union exists in sse.ts with all 17 interfaces, but the union type is never imported or applied outside sse.ts. In use-chat.ts, JSON.parse returns untyped `any`; the switch(data.type) block accesses fields without type narrowing via SSEEvent. The key link 'use-chat.ts imports SSE types for event parsing' is only half-met: CommissionItem is imported, but SSEEvent is not used as a type annotation at the parse site."
    artifacts:
      - path: "frontend/src/hooks/use-chat.ts"
        issue: "Line 211: `const data = JSON.parse(jsonStr)` — data is untyped any. SSEEvent union never used for narrowing."
      - path: "frontend/src/types/sse.ts"
        issue: "SSEEvent union type defined but imported/used by zero consumer files outside this module."
    missing:
      - "Cast JSON.parse result to SSEEvent (e.g. `const data = JSON.parse(jsonStr) as SSEEvent`) or add a type guard function so the discriminated union actually provides compile-time safety at the parse site"
      - "Alternatively, import SSEEvent in use-chat.ts and annotate the data variable — this fulfills the key link intent"
human_verification:
  - test: "Navigate to /valutazione, /ranking, /explorer in browser"
    expected: "Each old route immediately redirects to /evaluation, /rankings, /explore respectively with no flash of content"
    why_human: "Client-side router.replace behavior and redirect timing cannot be verified by static analysis"
  - test: "Submit a question in the chat interface"
    expected: "Streaming SSE events render correctly (progress steps, expert panel, citations, balance chart) with no runtime TypeScript errors in console"
    why_human: "SSE event parsing at runtime uses untyped JSON.parse; type correctness cannot be fully confirmed without running the app"
---

# Phase 3: Frontend Cleanup Verification Report

**Phase Goal:** The Next.js frontend has strict TypeScript (no `any`), English naming throughout, and clean component structure with no dead code
**Verified:** 2026-04-02T19:00:00Z
**Status:** gaps_found
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | tsc --noEmit compiles with zero errors | VERIFIED | Exit code 0, no output |
| 2 | grep for `: any` in src/ returns zero matches | VERIFIED | 0 matches; `as any` also 0 matches |
| 3 | Every SSE event payload has a typed interface in sse.ts | VERIFIED | 17 interfaces + SSEEvent union in frontend/src/types/sse.ts |
| 4 | SSEEvent union used for event parsing in use-chat.ts | FAILED | SSEEvent never imported; `JSON.parse` result is untyped at line 211 |
| 5 | Navigating to /valutazione/ranking/explorer redirects to new routes | VERIFIED | router.replace in all 3 stub files (line 8 each) |
| 6 | All code-level Italian variable names and comments are in English | VERIFIED | maggioranzaPercentage/opposizionePercentage gone; targeted comments translated |
| 7 | No dead code remains; every component folder has a barrel index.ts | VERIFIED | formatEventForFrontend/formatExpert/formatCitation deleted; addUserMessage removed; 5 barrel files created |

**Score:** 6/7 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `frontend/src/types/sse.ts` | SSE event discriminated union types | VERIFIED | 17 interfaces, SSEEvent union, CommissionItem — 160 lines |
| `frontend/src/types/chat.ts` | Updated chat types, no any, ChatHistoryItem | VERIFIED | ChatHistoryItem at line 160, StepResult at line 153, zero any |
| `frontend/src/app/evaluation/page.tsx` | Evaluation page (moved from valutazione) | VERIFIED | 1372 lines, full content |
| `frontend/src/app/rankings/page.tsx` | Rankings page (moved from ranking) | VERIFIED | 844 lines, full content |
| `frontend/src/app/explore/page.tsx` | Explore page (moved from explorer) | VERIFIED | 331 lines, full content |
| `frontend/src/app/valutazione/page.tsx` | Client-side redirect to /evaluation | VERIFIED | router.replace("/evaluation") at line 8 |
| `frontend/src/app/ranking/page.tsx` | Client-side redirect to /rankings | VERIFIED | router.replace("/rankings") at line 8 |
| `frontend/src/app/explorer/page.tsx` | Client-side redirect to /explore | VERIFIED | router.replace("/explore") at line 8 |
| `frontend/src/components/evaluation/index.ts` | Barrel export for evaluation components | VERIFIED | Exports RadarChart, HorizontalBarChart, ScoreDistribution, MetricCard, MiniMetricBars, ABComparisonChart, WinRateChart |
| `frontend/src/components/graph/index.ts` | Barrel export for graph components | VERIFIED | Exports GraphVisualizer |
| `frontend/src/components/search/index.ts` | Barrel export for search components | VERIFIED | Exports DeputySelector, GroupSelector, ResultsList, ResultDetailDialog |
| `frontend/src/components/settings/index.ts` | Barrel export for settings components | VERIFIED | Exports SettingsModal, RetrievalEditor, AuthorityEditor, GenerationEditor |
| `frontend/src/components/ui/index.ts` | Barrel export for shadcn/ui primitives | VERIFIED | 18 export lines covering 15 component families |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `frontend/src/hooks/use-chat.ts` | `frontend/src/types/sse.ts` | import SSE types for event parsing | PARTIAL | CommissionItem imported (line 15); SSEEvent union NOT imported; JSON.parse result at line 211 is untyped `any` |
| `frontend/src/components/shared/HistoryModal.tsx` | `frontend/src/types/chat.ts` | import ChatHistoryItem | VERIFIED | `import type { ChatHistoryItem } from "@/types"` at line 31; used in useState and onLoadChat prop |
| `frontend/src/components/layout/Sidebar.tsx` | `frontend/src/app/rankings/page.tsx` | href links updated from /ranking to /rankings | VERIFIED | 2 occurrences of /rankings at lines 144 and 254; zero occurrences of `"/ranking"` |
| `frontend/src/app/valutazione/page.tsx` | `frontend/src/app/evaluation/page.tsx` | router.replace redirect | VERIFIED | router.replace("/evaluation") present |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| FE-01 | 03-01 | Strict TypeScript — no `any` types, proper interfaces for all API responses | SATISFIED | tsc exits 0; 0 `: any` matches; 0 `as any` matches; sse.ts has full typed interfaces |
| FE-02 | 03-02 | Clean component structure — remove dead code, consistent naming | SATISFIED | formatEventForFrontend/formatExpert/formatCitation deleted from route.ts; addUserMessage removed from use-chat.ts; 5 barrel index.ts files created |
| FE-03 | 03-02 | English route paths and variable names where currently Italian | SATISFIED | /evaluation, /rankings, /explore routes created; redirects from old paths; maggioranzaPercentage → majorityPercentage; wire-format values preserved |

### Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| `frontend/src/hooks/use-chat.ts:211` | `const data = JSON.parse(jsonStr)` — data is `any`, SSEEvent union unused | Warning | SSE types defined but provide no compile-time safety at the parse site; runtime field access unchecked by tsc |
| `frontend/src/types/index.ts:2` | Italian docstring: "Tipi TypeScript centralizzati per l'applicazione" | Info | Not in plan scope; minor inconsistency with FE-03 intent |

### Human Verification Required

#### 1. Route Redirect Timing

**Test:** Navigate to `/valutazione`, `/ranking`, and `/explorer` in a browser
**Expected:** Each old path immediately replaces with the new route — /evaluation, /rankings, /explore — with no visible flash of content or broken layout
**Why human:** Client-side `useEffect` + `router.replace` timing and Next.js hydration behavior cannot be verified by static analysis

#### 2. Chat Streaming Integrity

**Test:** Submit a question in the chat interface and observe the full response flow
**Expected:** Progress steps appear, expert panel populates, citations render, balance chart shows — with no JavaScript errors in the browser console
**Why human:** The SSE parsing in use-chat.ts uses untyped `JSON.parse` (data is `any`); while tsc passes, runtime correctness of field access on SSE payloads must be verified empirically

### Gaps Summary

One gap blocks full goal achievement: the SSEEvent discriminated union exists as typed interfaces in `sse.ts` but is never applied at the parse site in `use-chat.ts`. The plan's intent — that SSE event payloads are typed — is partially met (interfaces exist) but the compile-time safety benefit is unrealized because `JSON.parse` at line 211 returns untyped `any` and the switch block accesses fields without type narrowing.

This is a low-severity gap: tsc already passes at zero errors, and the SSE contract is stable, so runtime behavior is correct. However, the goal of "strict TypeScript with proper interfaces for all API responses" (FE-01) is semantically incomplete if the interfaces are defined but not wired to the actual parse point.

The fix is minimal: annotate `const data = JSON.parse(jsonStr) as SSEEvent` (or add a type guard) in use-chat.ts. This would make the discriminated union actively protective and complete the key link.

One informational item: `frontend/src/types/index.ts` retains an Italian docstring ("Tipi TypeScript centralizzati per l'applicazione") that was outside the plan's file scope but is inconsistent with the FE-03 English naming goal.

---

_Verified: 2026-04-02T19:00:00Z_
_Verifier: Claude (gsd-verifier)_
