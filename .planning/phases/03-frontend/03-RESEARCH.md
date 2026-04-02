# Phase 3: Frontend - Research

**Researched:** 2026-04-02
**Domain:** Next.js 16 / TypeScript strict mode / App Router cleanup
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Route Renaming**
- `/valutazione` → `/evaluation` (the only Italian route)
- `/ranking` → `/rankings` (pluralize for consistency)
- `/explorer` → `/explore` (verb form)
- `/compass` → `/compass` (unchanged)
- `/search` → `/search` (unchanged)
- `/chat/[id]` → `/chat/[id]` (unchanged)
- **Redirect strategy:** Client-side redirect (`useEffect` + `router.replace`) in old route pages, NOT server-side 301
- All Sidebar links, internal navigation, and references must update to new paths

**TypeScript Strictness**
- **Zero `any`** — eliminate all 56 occurrences across 15 files
- For SSE parsing in `use-chat.ts` (14 `any`): define typed interfaces for every SSE event payload
- **Enable `strict: true`** in `tsconfig.json` — `strictNullChecks`, `noImplicitAny`, etc.
- `tsc --strict` must compile with zero errors after this phase

**Italian String Cleanup**
- **User-facing UI text STAYS in Italian** — labels, headings, button text, tooltips, error messages shown to users. The app is for Italian users.
- **Code-level Italian → English** — all variable names, function names, comments, docstrings must be in English

**Component Structure**
- **Keep current feature-based organization** — `chat/`, `evaluation/`, `graph/`, `layout/`, `search/`, `settings/`, `shared/`, `survey/`, `ui/`
- **Add barrel exports** (`index.ts`) to every component folder that doesn't have one
- **Remove dead code** — unused components, unused imports, unused variables
- Do NOT restructure into atoms/molecules/organisms or similar patterns

### Claude's Discretion
- Exact SSE event type interfaces (derive from `SSE_CONTRACT.md`)
- Which components have dead code to remove
- Exact barrel export contents per folder
- How to handle `strict: true` compilation errors beyond `any` removal
- Whether to add `// @ts-expect-error` comments for genuine external lib issues

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| FE-01 | Strict TypeScript — no `any` types, proper interfaces for all API responses | `strict: true` already in tsconfig; 50 `any` found across 15 files; SSE_CONTRACT.md provides exact payload shapes for typed interfaces |
| FE-02 | Clean component structure — remove dead code, consistent naming | 5 of 9 component folders lack `index.ts`; `addUserMessage` returned from hook but never used by any component; `formatEventForFrontend` defined but never called in route.ts |
| FE-03 | English route paths and variable names where currently Italian | Route `/valutazione` identified; Italian identifiers mapped: `commissioni`, `magg`/`opp`, `maggioranzaPercentage`/`opposizionePercentage`, `commList`/`commResult`/`topComm`; Italian docstrings in 5 files |
</phase_requirements>

---

## Summary

Phase 3 is a code-quality cleanup with zero behavioral changes. The TypeScript work is the heaviest task: `strict: true` is already set in `tsconfig.json` but the codebase has accumulated 50 `any` occurrences across 15 files, primarily because SSE event payloads were never given typed interfaces. The SSE_CONTRACT.md (frozen in Phase 2) provides the exact payload shapes — implementing those interfaces will eliminate the majority of `any` in `use-chat.ts`.

The Italian code cleanup is targeted: there are no Italian route segments beyond `/valutazione`, no Italian-named component files, and UI text intentionally stays Italian. The code-level Italian is concentrated in variable names (`commissioni`, `magg`/`opp`, `maggioranzaPercentage`/`opposizionePercentage`), a few comments in `route.ts` and `use-chat.ts`, and a handful of docstrings. The `BalanceMetrics` interface in `types/chat.ts` has Italian-named properties that cascade across 14 references — this is the single highest-impact Italian rename.

Component barrel exports are missing from exactly 5 folders (`evaluation/`, `graph/`, `search/`, `settings/`, `ui/`). Dead code is narrow: `addUserMessage` is exported from `useChat` but no component calls it; `formatEventForFrontend` is defined in `route.ts` but never invoked; the `route.ts` itself has a full `fallbackMockResponse` with embedded Italian comments and strings.

**Primary recommendation:** Work in three sequential waves: (1) define SSE interfaces + eliminate `any`, (2) rename Italian identifiers + translate comments, (3) routes + barrel exports + dead code.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Next.js | 16.1.4 | App Router, file-based routing | Already in use |
| React | 19.2.3 | UI rendering | Already in use |
| TypeScript | ^5 | Static typing | Already in use |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| next/navigation `useRouter` | built-in | Client-side redirect in old route files | Redirect strategy for renamed routes |

### Compilation Check
```bash
cd /Users/mirkotritella/Desktop/ParliamentRAG/frontend && npx tsc --noEmit
```
This is the authoritative zero-error check that must pass after all `any` removals.

---

## Architecture Patterns

### Recommended Project Structure (unchanged)
```
frontend/src/
├── app/
│   ├── chat/[id]/         # unchanged route
│   ├── compass/           # unchanged route
│   ├── evaluate/          # NEW — was valutazione/
│   ├── explore/           # NEW — was explorer/
│   ├── rankings/          # NEW — was ranking/
│   ├── search/            # unchanged route
│   ├── valutazione/       # KEEP — add redirect page only
│   ├── explorer/          # KEEP — add redirect page only
│   ├── ranking/           # KEEP — add redirect page only
│   └── api/chat/          # Next.js API route (proxy)
├── components/
│   ├── chat/index.ts      # exists
│   ├── evaluation/index.ts  # ADD
│   ├── graph/index.ts     # ADD
│   ├── layout/index.ts    # exists
│   ├── search/index.ts    # ADD
│   ├── settings/index.ts  # ADD
│   ├── shared/index.ts    # exists
│   ├── survey/index.ts    # exists
│   └── ui/index.ts        # ADD (but do NOT modify ui/*.tsx files)
├── hooks/
│   └── use-chat.ts
├── types/
│   ├── chat.ts
│   ├── api.ts             # StreamEventType enum needs updating
│   ├── evaluation.ts
│   ├── survey.ts
│   └── sse.ts             # NEW — SSE payload interfaces
└── lib/
    ├── api.ts
    ├── graph-api.ts
    └── ...
```

### Pattern 1: Client-Side Redirect for Renamed Routes
**What:** Old route pages become thin redirect-only files that immediately push to the new path.
**When to use:** Every renamed route (`/valutazione`, `/ranking`, `/explorer`).
**Example:**
```typescript
// src/app/valutazione/page.tsx  (after rename — this file becomes the redirect)
"use client";
import { useEffect } from "react";
import { useRouter } from "next/navigation";

export default function ValutazioneRedirect() {
  const router = useRouter();
  useEffect(() => {
    router.replace("/evaluation");
  }, [router]);
  return null;
}
```

### Pattern 2: SSE Event Discriminated Union
**What:** Each SSE event type gets its own payload interface, combined into a discriminated union on `type`.
**When to use:** Typed parsing in `use-chat.ts` to eliminate all SSE-related `any`.
**Example (derived from SSE_CONTRACT.md):**
```typescript
// src/types/sse.ts

export interface SSEWaitingEvent {
  type: "waiting";
  queue_position?: number;
  ahead_count?: number;
  active_count?: number;
  elapsed_seconds?: number;
  message?: string;
}

export interface SSEProgressEvent {
  type: "progress";
  step: number;
  total?: number;
  message: string;
}

export interface SSEExpertsChatEvent {
  type: "experts";
  experts: import("./chat").Expert[];  // chat.py pipeline
}

export interface SSEExpertsQueryEvent {
  type: "experts";
  data: import("./chat").Expert[];     // query.py pipeline
}

export interface SSEChunkChatEvent {
  type: "chunk";
  content: string;                     // chat.py uses "content" key
}

export interface SSEChunkQueryEvent {
  type: "chunk";
  data: string;                        // query.py uses "data" key
}

export interface SSECitationsChatEvent {
  type: "citations";
  citations: import("./chat").Citation[];
}

export interface SSECitationsQueryEvent {
  type: "citations";
  data: import("./chat").Citation[];
}

export interface SSECommissioniEvent {
  type: "commissioni";
  commissioni: CommissionItem[];
}

export interface CommissionItem {
  name: string;
  nome?: string;  // legacy field from backend
  score?: number;
  matched_keywords?: string[];
  categories?: string[];
  url?: string;
}

export interface SSEBalanceEvent {
  type: "balance";
  maggioranza_percentage: number;
  opposizione_percentage: number;
  bias_score: number;
}

export interface SSECompassChatEvent {
  type: "compass";
  meta: import("../components/chat/CompassCard").CompassData["meta"];
  axes: import("../components/chat/CompassCard").CompassData["axes"];
  groups: import("../components/chat/CompassCard").CompassData["groups"];
  scatter_sample: import("../components/chat/CompassCard").CompassData["scatter_sample"];
}

export interface SSECompassQueryEvent {
  type: "compass";
  data: import("../components/chat/CompassCard").CompassData;
}

export interface SSETopicStatsEvent {
  type: "topic_stats";
  intervention_count: number;
  speaker_count: number;
  first_date: string;
  last_date: string;
  speakers_detail: import("./chat").SpeakerDetail[];
  interventions_detail: import("./chat").InterventionDetail[];
  sessions_detail: import("./chat").SessionDetail[];
}

export interface SSECitationDetailsEvent {
  type: "citation_details";
  citations: import("./chat").Citation[];
}

export interface SSEHQVariantsEvent {
  type: "hq_variants";
  variants: import("./chat").HQVariant[];
}

export interface SSECompleteEvent {
  type: "complete";
  metadata?: {
    timing?: Record<string, number>;
    dense_channel_count?: number;
    graph_channel_count?: number;
    [key: string]: unknown;
  };
}

export interface SSEErrorEvent {
  type: "error";
  message: string;
}

export type SSEEvent =
  | SSEWaitingEvent
  | SSEProgressEvent
  | SSEExpertsChatEvent
  | SSEExpertsQueryEvent
  | SSEChunkChatEvent
  | SSEChunkQueryEvent
  | SSECitationsChatEvent
  | SSECitationsQueryEvent
  | SSECommissioniEvent
  | SSEBalanceEvent
  | SSECompassChatEvent
  | SSECompassQueryEvent
  | SSETopicStatsEvent
  | SSECitationDetailsEvent
  | SSEHQVariantsEvent
  | SSECompleteEvent
  | SSEErrorEvent;
```

### Anti-Patterns to Avoid
- **`[key: string]: any` index signature:** `Citation` in `types/chat.ts` line 26 has this escape hatch — remove it and add explicit optional fields for every known key instead.
- **`details?: any` in `StepResult`:** Replace with `details?: StepResultDetails` typed per step type.
- **`loadChat(historyData: any)`:** The history API response has a known shape — define `ChatHistoryItem` and use it.
- **`formatEventForFrontend` dead code:** This function in `route.ts` is never called — delete it along with `formatExpert` and `formatCitation`. The route simply proxies bytes unchanged.

---

## Complete `any` Map

Every `any` occurrence found, with recommended replacement:

### `frontend/src/types/chat.ts` (2 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 26 | `[key: string]: any` (index sig on `Citation`) | Remove index sig; add explicit optional fields |
| 152 | `details?: any` (on `StepResult`) | `details?: StepResultDetails` (new union type per step) |

### `frontend/src/types/survey.ts` (1 occurrence)
| Line | Current | Replacement |
|------|---------|-------------|
| 215 | `baseline_experts?: any[]` | `baseline_experts?: Expert[]` (import from `./chat`) |

### `frontend/src/hooks/use-chat.ts` (11 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 178 | `let commissioni: any[]` | `let committeeMatches: CommissionItem[]` (from `sse.ts`) |
| 180 | `Map<number, { step...; details?: any }>` | `Map<number, StepResult>` (existing type) |
| 261 | `Map<number, { step...; details?: any }>` | `Map<number, StepResult>` |
| 297 | `commList.map((c: any) => ...)` | `commList.map((c: CommissionItem) => ...)` |
| 484 | `Map<number, any>` | `Map<number, StepResult>` |
| 669 | `loadChat(historyData: any)` | `loadChat(historyData: ChatHistoryItem)` (new interface) |
| 704 | `{ step: number; label: string; result: string; details?: any }[]` | `StepResult[]` |
| 708 | `historyData.citations.map((c: any) => ...)` | typed via `ChatHistoryItem.citations: Citation[]` |
| 717 | `historyData.experts.filter((e: any) => ...)` | typed via `ChatHistoryItem.experts: Expert[]` |
| 718 | `historyData.experts.filter((e: any) => ...)` | same as above |
| 719 | `historyData.experts.slice(0,3).map((e: any) => ...)` | same as above |

### `frontend/src/lib/api.ts` (1 occurrence)
| Line | Current | Replacement |
|------|---------|-------------|
| 45 | `function mapRawToConfig(raw: any)` | `function mapRawToConfig(raw: unknown)` + runtime check or cast to `SystemConfig` directly |

### `frontend/src/components/chat/CompassCard.tsx` (2 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 33 | `stats: any` (inside `CompassData.groups[]`) | `stats: Record<string, number>` or `CompassGroupStats` interface |
| 166 | `style?: any` in `AxisLabel` props | `style?: React.CSSProperties` |

### `frontend/src/components/chat/MessageBubble.tsx` (1 occurrence)
| Line | Current | Replacement |
|------|---------|-------------|
| 841 | `{ variant: any }` on `VariantCard` | `{ variant: HQVariant }` (import from `types/chat`) |

### `frontend/src/components/graph/GraphVisualizer.tsx` (8 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 16 | `data: any[]` (prop) | `data: GraphRecord[]` (new interface) |
| 20 | `{ nodes: any[]; links: any[] }` | `{ nodes: GraphNode[]; links: GraphLink[] }` |
| 24 | `useRef<any>(null)` | `useRef<ForceGraphMethods \| null>(null)` from `react-force-graph-2d` |
| 27 | `useState<any>(null)` | `useState<GraphNode \| null>(null)` |
| 45 | `processItem = (item: any)` | `processItem = (item: GraphRecord)` |
| 205 | `handleNodeClick = (node: any)` | `handleNodeClick = (node: NodeObject)` from `react-force-graph-2d` |
| 248 | `nodeCanvasObject={(node: any, ctx, globalScale)` | `(node: NodeObject, ctx: CanvasRenderingContext2D, globalScale: number)` |
| 273 | `nodePointerAreaPaint={(node: any, color, ctx)` | `(node: NodeObject, color: string, ctx: CanvasRenderingContext2D)` |

### `frontend/src/components/shared/HistoryModal.tsx` (2 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 35 | `onLoadChat?: (chat: any) => void` | `onLoadChat?: (chat: ChatHistoryItem) => void` |
| 51 | `useState<any[]>([])` | `useState<ChatHistoryItem[]>([])` |

### `frontend/src/components/shared/ProgressIndicator.tsx` (1 occurrence)
| Line | Current | Replacement |
|------|---------|-------------|
| 57 | `details?: any` in `StepResultDetails` component props | `details?: StepResultDetails` (same new union) |

### `frontend/src/components/settings/SettingsModal.tsx` (1 occurrence)
| Line | Current | Replacement |
|------|---------|-------------|
| 121 | `let parsed: any` | `let parsed: SystemConfig` (import from `lib/api`) |

### `frontend/src/components/survey/SurveyModal.tsx` (9 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 72 | `citations: any[]` (inside local interface) | `citations: Citation[]` |
| 74 | `balance: any` | `balance: BalancePayload` (or `BalanceMetrics`) |
| 75 | `compass: any` | `compass: CompassData \| null` |
| 833 | `{ answer?: string; experts?: Expert[]; citations?: any[] }` | `{ answer?: string; experts?: Expert[]; citations?: Citation[] }` |
| 857 | `(chatData.citations ?? []) as any[]` | `chatData.citations ?? []` (already typed) |
| 875 | `.map((c: any) => ...)` | `.map((c: Citation) => ...)` |
| 894 | `.map((c: any) => ...)` | `.map((c: Citation) => ...)` |
| 1079 | `err: any` | `err: unknown` + `instanceof Error` guard |
| 1109 | `err: any` | `err: unknown` + `instanceof Error` guard |

### `frontend/src/app/ranking/page.tsx` (2 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 126 | `data.deputies.map((d: any) => ...)` | `data.deputies.map((d: RawDeputyApiItem) => ...)` (new local interface) |
| 134 | `catch (e: any)` | `catch (e: unknown)` + `instanceof Error` guard |

### `frontend/src/app/explorer/page.tsx` (4 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 33 | `useState<any>(null)` (schema) | `useState<GraphSchema \| null>(null)` |
| 34 | `useState<any>(null)` (stats) | `useState<GraphStats \| null>(null)` |
| 36 | `useState<any[]>([])` (results) | `useState<GraphQueryResult[]>([])` |
| 277 | `Object.values(row).map((val: any, j)` | `Object.values(row).map((val: unknown, j)` |

### `frontend/src/app/compass/page.tsx` (4 occurrences)
| Line | Current | Replacement |
|------|---------|-------------|
| 62 | `groups: any[]` | `groups: CompassData["groups"]` |
| 63 | `scatter_sample: any[]` | `scatter_sample: CompassData["scatter_sample"]` |
| 99 | `catch (e: any)` | `catch (e: unknown)` + `instanceof Error` guard |
| 402 | `<CompassCard data={compassData as any} />` | Fix type of `compassData` state to match `CompassData` prop |

### `frontend/src/app/valutazione/page.tsx` (1 occurrence)
| Line | Current | Replacement |
|------|---------|-------------|
| 1236 | `(item.human as any)[dim]` | Use `keyof SurveyResponse` with a type-safe lookup |

---

## Italian Code-Level Identifiers Map

**Rule:** Variable names, function names, type property names, and comments in code must be English. UI strings in JSX/templates are excluded.

### Type-Level Italian (cascading impact — fix types first)

| File | Current | Replacement | Impact |
|------|---------|-------------|--------|
| `types/chat.ts:134` | `maggioranzaPercentage` (on `BalanceMetrics`) | `majorityPercentage` | 14 references across 5 files |
| `types/chat.ts:135` | `opposizionePercentage` (on `BalanceMetrics`) | `oppositionPercentage` | 14 references across 5 files |
| `types/chat.ts:121` | `// Metadati opzionali per le risposte dell'assistente` (comment) | `// Optional metadata for assistant responses` | comment-only |
| `types/chat.ts:136` | `// -1 = tutto opposizione, 0 = bilanciato, 1 = tutto maggioranza` (comment) | `// -1 = all opposition, 0 = balanced, 1 = all majority` | comment-only |
| `types/api.ts:2` | `* Tipi per le API` (docstring) | `* API types` | docstring-only |
| `types/chat.ts:2` | `* Tipi per il sistema di chat` (docstring) | `* Types for the chat system` | docstring-only |
| `types/survey.ts:299` | `// Authority dimensions — Autorità Esperti category` | `// Authority dimensions — Expert Authority category` | comment-only |

### Variable-Level Italian in `use-chat.ts`

| Line | Current | Replacement | Note |
|------|---------|-------------|------|
| 178 | `let commissioni: any[]` | `let committeeMatches: CommissionItem[]` | SSE field name `commissioni` is a frozen API key, stays on wire |
| 294 | `const commList` | `const committeeList` | |
| 295 | `commissioni = commList` | `committeeMatches = committeeList` | |
| 296 | `updateLastAssistantMessage({ commissioni: [...commList] })` | `updateLastAssistantMessage({ committeeMatches: [...committeeList] })` | Also requires updating `Message.commissioni` in types |
| 297 | `const commNames` | `const committeeNames` | |
| 299 | `const topComm` | `const topCommittee` | |
| 300 | `const commResult` | `const committeeResult` | |
| 324 | `const magg` | `const majorityCount` | |
| 325 | `const opp` | `const oppositionCount` | |
| 374 | `maggioranzaPercentage: data.maggioranza_percentage` | `majorityPercentage: data.maggioranza_percentage` | after BalanceMetrics rename |
| 375 | `opposizionePercentage: data.opposizione_percentage` | `oppositionPercentage: data.opposizione_percentage` | after BalanceMetrics rename |
| 214 | `// Aggiungi al buffer e processa solo messaggi completi` | `// Append to buffer and process complete messages only` | comment |
| 217 | `buffer = messages.pop() || ""; // Mantieni l'ultimo (potenzialmente incompleto)` | `// Keep the last (potentially incomplete) chunk` | comment |

### Variable-Level Italian in `app/api/chat/route.ts`

| Line | Current | Replacement |
|------|---------|-------------|
| 4 | `* API Route per la chat RAG` (docstring) | `* Chat RAG API route` |
| 6-8 | Italian docstring body | English translation |
| 24 | `// Proxy verso il backend FastAPI` | `// Proxy request to FastAPI backend` |
| 38 | `// Stream diretto dal backend` | `// Stream directly from backend` |
| 48 | `let buffer = ""; // Buffer per messaggi parziali` | `// Buffer for partial messages` |
| 55-78 | Italian inline comments throughout | Translate to English |
| 109 | `// Fallback: usa mock data se il backend non è disponibile` | `// Fallback: use mock data if backend is unavailable` |
| 115 | `* Formatta gli eventi dal backend per il frontend` (dead code docstring) | Delete (dead code) |
| 182-205 | `formatEventForFrontend`, `formatExpert`, `formatCitation` functions | **Delete entirely** — never called |
| 231 | `* Fallback con mock data quando il backend non è disponibile` | `* Fallback with mock data when backend is unavailable` |
| 241 | `// Avvisa che stiamo usando il fallback` | `// Warn that we're using fallback mode` |
| 244 | `// Simula i 6 step del pipeline RAG` | `// Simulate the 6 pipeline steps` |
| 259 | `// Simula streaming della risposta` | `// Simulate streaming response` |
| 283 | `// Completa` | `// Complete` |
| 287 | `"Errore durante l'elaborazione"` | This is an error string in mock — it will never show to real users; replace with `"Error during processing"` (not a user-facing UI string) |

### Variable-Level Italian in `types/chat.ts`

| Line | Current | Replacement |
|------|---------|-------------|
| 124 | `commissioni?: Array<{ nome: string; ... }>` on `Message` | Rename to `committeeMatches?: CommissionItem[]`; remove `nome` (it's the Italian SSE wire name) |

### Variable-Level Italian in `app/ranking/page.tsx`

| Line | Current | Replacement |
|------|---------|-------------|
| 54 | `type CoalitionFilter = "all" \| "maggioranza" \| "opposizione"` | `type CoalitionFilter = "all" \| "majority" \| "opposition"` |
| 97 | `const [coalitionFilter, setCoalitionFilter] = useState<CoalitionFilter>("all")` | unchanged (already English) |
| 307 | Filter values `"maggioranza"` / `"opposizione"` in JSX | Update to `"majority"` / `"opposition"` |

**IMPORTANT:** `maggioranza` and `opposizione` appear as **wire values** in the `Expert.coalition` field and `BalanceMetrics` SSE payload — these are frozen backend contract values from SSE_CONTRACT.md. The `coalition: "maggioranza"` comparison strings in filter predicates (`e.coalition === "maggioranza"`) must remain as-is because they match the frozen SSE wire values. Only local TypeScript type aliases are renamed.

### Variable-Level Italian in `config/index.ts`

| Line | Current | Replacement |
|------|---------|-------------|
| 2-3 | `* Configurazione centralizzata dell'applicazione / Modifica questi valori...` | English docstring |

### Variable-Level Italian in `lib/graph-api.ts`

| Line | Current | Replacement |
|------|---------|-------------|
| 32 | `"Operazione di scrittura non consentita..."` error message | This surfaces to the browser console only (thrown in `executeCypherQuery` which is a code path, not shown in UI) → `"Write operations are not allowed. The Graph Explorer is read-only."` |

### Variable-Level Italian in `lib/api.ts`

| Line | Current | Replacement |
|------|---------|-------------|
| 78 | `\`Errore nel caricamento config: ...\`` | `\`Failed to load config: ...\`` (thrown error, not UI text) |

---

## Route Rename Map

### Files to Create (new route destinations)
| New Path | New File |
|----------|----------|
| `/evaluation` | `frontend/src/app/evaluation/page.tsx` (move content from `valutazione/page.tsx`) |
| `/rankings` | `frontend/src/app/rankings/page.tsx` (move content from `ranking/page.tsx`) |
| `/explore` | `frontend/src/app/explore/page.tsx` (move content from `explorer/page.tsx`) |

### Files to Replace with Redirects (old paths preserved for compatibility)
| Old Path | File After Phase |
|----------|-----------------|
| `/valutazione` | Replace `frontend/src/app/valutazione/page.tsx` with redirect to `/evaluation` |
| `/ranking` | Replace `frontend/src/app/ranking/page.tsx` with redirect to `/rankings` |
| `/explorer` | Replace `frontend/src/app/explorer/page.tsx` with redirect to `/explore` |

### Sidebar Links to Update (`components/layout/Sidebar.tsx`)
| Current | New |
|---------|-----|
| `href: "/ranking"`, `pathname === "/ranking"`, `navTo("/ranking")` (×4 occurrences — desktop + mobile) | `/rankings` |
| No evaluation/explore links currently in Sidebar | No change needed — `/valutazione` and `/explorer` are not in the sidebar nav |

### Internal Navigation Audit
- `Sidebar.tsx` has `/ranking` in 4 places (2 mobile, 2 desktop). No `/valutazione` or `/explorer` links.
- `/valutazione/page.tsx` and `/explorer/page.tsx` do not cross-link to each other.
- No `<Link href="/ranking">`, `<Link href="/valutazione">`, or `router.push` calls found in component files beyond Sidebar.

---

## Dead Code Map

### Confirmed Dead Code to Delete

| File | Dead Item | Evidence |
|------|-----------|---------|
| `app/api/chat/route.ts` lines 117-226 | `formatEventForFrontend`, `formatExpert`, `formatCitation` functions | Never called — route simply proxies bytes |
| `hooks/use-chat.ts` `addUserMessage` | Defined and returned from hook but zero component imports it | `grep addUserMessage *.tsx` returns 0 results |
| `hooks/use-chat.ts` line 343 | Empty `else {}` block after experts payload check | Explicitly empty |
| `hooks/use-chat.ts` line 368 | Empty `else {}` block after citations payload check | Explicitly empty |
| `hooks/use-chat.ts` line 512-513 | Empty `if (data.metadata?.timing)` block | `const timing = data.metadata.timing;` unused |
| `hooks/use-chat.ts` line 475-479 | `unmatchedLinks` logging block with empty `if/else` branches | Both branches are empty |

### Possibly Dead Code — Verify Before Deleting
| File | Item | Note |
|------|------|------|
| `app/api/chat/route.ts` `fallbackMockResponse` | Called in catch block | May be intentional dev fallback; ask or keep |
| `use-chat.ts` `addUserMessage` | In hook return | Not used by components but is exported — confirm no external caller |

---

## Barrel Export Map

### Folders That Need `index.ts` Added

**`components/evaluation/index.ts`:**
```typescript
export { EvaluationCharts } from "./EvaluationCharts";
```

**`components/graph/index.ts`:**
```typescript
export { GraphVisualizer } from "./GraphVisualizer";
```

**`components/search/index.ts`:**
```typescript
export { DeputySelector } from "./DeputySelector";
export type { Deputy } from "./DeputySelector";
export { GroupSelector } from "./GroupSelector";
export { ResultsList } from "./ResultsList";
export type { SearchResultItem } from "./ResultsList";
export { ResultDetailDialog } from "./ResultDetailDialog";
```

**`components/settings/index.ts`:**
```typescript
export { SettingsModal } from "./SettingsModal";
export { GraphicalEditors } from "./GraphicalEditors";
```

**`components/ui/index.ts`:**
```typescript
// Re-export all shadcn/ui primitives used in this project
// Do NOT modify any ui/*.tsx files
export { Button } from "./button";
export { Badge } from "./badge";
export { Card, CardContent, CardHeader, CardTitle } from "./card";
// ... (enumerate from actual files in ui/)
```

---

## TypeScript Configuration — Current State

`frontend/tsconfig.json` already has `"strict": true`. This enables all strict sub-flags including:
- `strictNullChecks`
- `noImplicitAny`
- `strictFunctionTypes`
- `strictBindCallApply`
- `strictPropertyInitialization`

**Finding:** `strict: true` is already set. The 50 `any` occurrences are explicit `any` type annotations — they still compile under `strict: true` because explicit `any` is allowed even with `noImplicitAny`. Removing them requires replacing with specific types. The `tsc --noEmit` check is the gating test.

**Additional flag to consider:** `"noUncheckedIndexedAccess": true` — forces `T | undefined` for array indexing. This is NOT part of `strict: true`. Adding it would cause many new errors. **Do not add it** unless CONTEXT.md is updated. Stick to zero `any` as the goal.

---

## Common Pitfalls

### Pitfall 1: Breaking SSE Wire Field Names
**What goes wrong:** Renaming `commissioni` variable leads to renaming the field in the SSE event parse (`case "commissioni":`) or the payload key.
**Why it happens:** Developer conflates local variable name with the SSE event type/key.
**How to avoid:** Only rename the local variable. `data.commissioni` and `case "commissioni":` reference the wire protocol — these stay as-is per SSE_CONTRACT.md.
**Warning signs:** Backend SSE events stop being parsed by the frontend.

### Pitfall 2: `coalition` Wire Values Are Not Italian Identifiers
**What goes wrong:** Renaming `CoalitionFilter` type values from `"maggioranza"/"opposizione"` to `"majority"/"opposition"` and then also changing the comparison `e.coalition === "maggioranza"` — which breaks expert filtering.
**Why it happens:** `maggioranza`/`opposizione` appear both as local type union values AND as frozen wire values in `Expert.coalition` (from backend).
**How to avoid:** Only rename the local `CoalitionFilter` type union. Never change string comparisons against `Expert.coalition` — these match frozen backend values. Check SSE_CONTRACT.md: `"coalition": "maggioranza"` is the wire value.

### Pitfall 3: `strict: true` Already Enabled — Extra Errors Are Real
**What goes wrong:** Developer adds strict flag thinking it was missing, then sees 200 new tsc errors and reverts.
**Why it happens:** The flag is already there; errors come from actual type gaps surfaced by removing `any`.
**How to avoid:** Know the baseline: run `tsc --noEmit` BEFORE changes to count existing errors. Then remove `any` one file at a time, keeping tsc clean.

### Pitfall 4: `BalanceMetrics` Rename Cascade
**What goes wrong:** Renaming `maggioranzaPercentage` → `majorityPercentage` in the type interface but missing one of the 14 references.
**Why it happens:** References span 5 files (`types/chat.ts`, `use-chat.ts`, `app/api/chat/route.ts`, `MessageBubble.tsx`, `SurveyModal.tsx`).
**How to avoid:** Use TypeScript compiler errors as the complete checklist — after renaming the interface field, let `tsc --noEmit` enumerate all remaining references.

### Pitfall 5: `CompassData` Cross-Import Cycle
**What goes wrong:** Moving `CompassData` type from `components/chat/CompassCard.tsx` to `types/sse.ts` creates an import cycle if `CompassCard` then imports from `types/`.
**Why it happens:** `CompassData` is currently defined in a component file, not a types file.
**How to avoid:** Keep `CompassData` where it is (`CompassCard.tsx`). In `types/sse.ts`, import it with `import type { CompassData } from "@/components/chat/CompassCard"`.

### Pitfall 6: New Route Folders Clobber Content Before Redirects
**What goes wrong:** Developer creates `app/evaluation/page.tsx` first, but old `app/valutazione/page.tsx` still contains the real content — both exist simultaneously and one overwrites the other.
**How to avoid:** Sequence matters: (1) create new route folder with moved content, (2) THEN replace old route file with redirect.

---

## Code Examples

### Redirect Page Pattern
```typescript
// Source: Next.js App Router docs (useRouter + replace pattern)
"use client";
import { useEffect } from "react";
import { useRouter } from "next/navigation";

export default function LegacyRouteRedirect() {
  const router = useRouter();
  useEffect(() => {
    router.replace("/new-path");
  }, [router]);
  return null; // Render nothing — redirect is instant
}
```

### `catch (e: unknown)` Pattern
```typescript
// Replaces: catch (e: any)
} catch (e: unknown) {
  const message = e instanceof Error ? e.message : "Unknown error";
  setError(message);
}
```

### SSE Event Narrowing in `use-chat.ts`
```typescript
// After importing SSEEvent from types/sse.ts:
const data = JSON.parse(jsonStr) as SSEEvent;

switch (data.type) {
  case "commissioni":
    // data is narrowed to SSECommissioniEvent here
    const committeeList = data.commissioni;
    break;
  case "experts":
    // data.experts exists for chat.py; data.data exists for query.py
    const expertsPayload = "experts" in data ? data.experts : data.data;
    break;
}
```

### Typed `ForceGraph2D` ref
```typescript
// react-force-graph-2d exports ForceGraphMethods
import ForceGraph2D, { type ForceGraphMethods, type NodeObject, type LinkObject } from "react-force-graph-2d";

const fgRef = useRef<ForceGraphMethods | null>(null);
```

---

## State of the Art

| Old Approach | Current Approach | Impact |
|--------------|------------------|--------|
| `catch (e: any)` | `catch (e: unknown)` (TS 4.0+) | Forces explicit type narrowing |
| Explicit `any` as escape hatch | Named `unknown` + type guards | Prevents type-unsafe operations |
| `strict: true` missing | Already enabled in this project | No tsconfig change needed |

---

## Validation Architecture

`nyquist_validation: true` — section included.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None installed — no test runner found |
| Config file | None — no jest.config, vitest.config, etc. |
| Quick run command | `cd /Users/mirkotritella/Desktop/ParliamentRAG/frontend && npx tsc --noEmit` |
| Full suite command | `cd /Users/mirkotritella/Desktop/ParliamentRAG/frontend && next build` |

**Note:** The frontend has no test framework installed (no Jest, Vitest, Playwright, or Cypress in `package.json` devDependencies). TypeScript compilation (`tsc --noEmit`) is the primary automated verification. Next.js build (`next build`) is the integration check.

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FE-01 | Zero `any` types, strict TypeScript compiles | tsc compile | `npx tsc --noEmit` | ✅ (tsconfig.json) |
| FE-02 | Dead code removed, barrel exports work | build check | `next build` | ✅ |
| FE-03 | Routes renamed, old routes redirect | smoke/manual | `next build` + manual browser check | ✅ |

### Sampling Rate
- **Per task commit:** `npx tsc --noEmit` (< 10 seconds)
- **Per wave merge:** `next build`
- **Phase gate:** Zero `tsc --noEmit` errors before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `src/types/sse.ts` — new file for SSE payload interfaces (Wave 0 creation, covers FE-01 SSE typing)
- [ ] `src/app/evaluation/page.tsx` — new route destination (Wave 0 skeleton, covers FE-03)
- [ ] `src/app/rankings/page.tsx` — new route destination
- [ ] `src/app/explore/page.tsx` — new route destination

---

## Open Questions

1. **`addUserMessage` truly unused?**
   - What we know: grep finds zero `.tsx` files importing it; it IS in the hook's return value.
   - What's unclear: Could an external consumer outside `src/` be using it?
   - Recommendation: Remove from hook return value; if build breaks, restore.

2. **`CoalitionFilter` values — rename or keep Italian?**
   - What we know: `"maggioranza"/"opposizione"` are also the frozen `Expert.coalition` wire values from the backend.
   - What's unclear: The local type alias values are used in comparisons that also match wire values.
   - Recommendation: Rename the type alias values to English (`"majority"/"opposition"`); keep wire comparisons against `Expert.coalition` unchanged (they compare against the API response, not the local type).

3. **`fallbackMockResponse` in `route.ts` — dead or dev tool?**
   - What we know: It is called in the `catch` block when backend is unreachable.
   - What's unclear: Whether this intentional dev fallback should be preserved or removed.
   - Recommendation: Keep but translate Italian strings inside it to English (covers FE-03 without breaking dev flow).

4. **`react-force-graph-2d` TypeScript types**
   - What we know: The library is in dependencies; 8 `any` occurrences in `GraphVisualizer.tsx` relate to it.
   - What's unclear: Whether `@types/react-force-graph-2d` exists or types ship with the package.
   - Recommendation: Check `node_modules/react-force-graph-2d/dist/react-force-graph-2d.d.ts` during implementation. If types are incomplete, use `// @ts-expect-error` with a comment explaining the external lib gap (per CONTEXT.md discretion).

---

## Sources

### Primary (HIGH confidence)
- Direct codebase inspection — `grep -rn "\bany\b"` full scan of `frontend/src`
- `frontend/tsconfig.json` — read directly
- `backend/docs/SSE_CONTRACT.md` — read directly (frozen Phase 2 artifact)
- `frontend/src/hooks/use-chat.ts` — read in full
- `frontend/src/components/layout/Sidebar.tsx` — read in full
- `frontend/src/types/chat.ts`, `api.ts`, `evaluation.ts`, `survey.ts` — read in full
- `frontend/package.json` — read directly

### Secondary (MEDIUM confidence)
- Next.js App Router docs pattern for `useRouter().replace()` client-side redirects — well-established pattern from Next.js 13+

### Tertiary (LOW confidence)
- None

---

## Metadata

**Confidence breakdown:**
- `any` map: HIGH — direct grep enumeration of every occurrence
- Italian identifier map: HIGH — direct code inspection with context
- Route rename scope: HIGH — Sidebar + filesystem confirmed
- Barrel export gaps: HIGH — `ls` + `index.ts` check on all 9 folders
- Dead code: HIGH for `formatEventForFrontend` (never called); MEDIUM for `addUserMessage` (no component consumer found but not exhaustively verified)
- SSE interfaces: HIGH — derived directly from frozen SSE_CONTRACT.md
- react-force-graph-2d typing: MEDIUM — depends on library type exports, needs verification at implementation time

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable codebase; no expected external churn)
