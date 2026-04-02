# Phase 3: Frontend - Context

**Gathered:** 2026-04-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Clean the Next.js frontend codebase: enable strict TypeScript (zero `any`), rename Italian routes to English with client-side redirects, translate all code-level Italian (variables, comments) to English while keeping user-facing UI text in Italian, remove dead code, and add barrel exports to all component folders.

</domain>

<decisions>
## Implementation Decisions

### Route Renaming

- `/valutazione` → `/evaluation` (the only Italian route)
- `/ranking` → `/rankings` (pluralize for consistency)
- `/explorer` → `/explore` (verb form)
- `/compass` → `/compass` (unchanged)
- `/search` → `/search` (unchanged)
- `/chat/[id]` → `/chat/[id]` (unchanged)
- **Redirect strategy:** Client-side redirect (`useEffect` + `router.replace`) in old route pages, NOT server-side 301
- All Sidebar links, internal navigation, and references must update to new paths

### TypeScript Strictness

- **Zero `any`** — eliminate all 56 occurrences across 15 files
- For SSE parsing in `use-chat.ts` (14 `any`): define typed interfaces for every SSE event payload
- **Enable `strict: true`** in `tsconfig.json` — `strictNullChecks`, `noImplicitAny`, etc.
- `tsc --strict` must compile with zero errors after this phase

### Italian String Cleanup

- **User-facing UI text STAYS in Italian** — labels, headings, button text, tooltips, error messages shown to users. The app is for Italian users.
- **Code-level Italian → English** — all variable names, function names, comments, docstrings must be in English
- This matches the backend convention established in Phase 2

### Component Structure

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

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Frontend source
- `frontend/src/hooks/use-chat.ts` — SSE parsing with 14 `any` occurrences (most impactful file)
- `frontend/src/types/` — Existing type definitions (chat.ts, api.ts, evaluation.ts, survey.ts)
- `frontend/src/app/valutazione/page.tsx` — Route being renamed
- `frontend/tsconfig.json` — TypeScript config to update

### SSE contract (from Phase 2)
- `backend/docs/SSE_CONTRACT.md` — Frozen SSE event types and payload shapes (source of truth for TypeScript interfaces)

### Phase 2 decisions
- `.planning/phases/02-backend/02-CONTEXT.md` — SSE payload field names frozen as snake_case

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `frontend/src/types/` — Already has chat.ts, api.ts, evaluation.ts, survey.ts type files
- `frontend/src/components/ui/` — shadcn/ui components (do not modify)
- `frontend/src/lib/utils.ts` — `cn()` utility for Tailwind class merging

### Established Patterns
- Next.js App Router with file-based routing
- Hooks in `src/hooks/` with `use-` prefix (kebab-case files)
- Components in PascalCase `.tsx` files
- API client functions in `src/lib/` (api.ts, evaluation-api.ts, survey-api.ts, graph-api.ts)
- shadcn/ui for UI primitives — do not modify `components/ui/`

### Integration Points
- `use-chat.ts` parses SSE events from backend — types must match SSE_CONTRACT.md
- `src/app/api/` contains Next.js API routes that proxy to backend
- `src/config/index.ts` has backend URL configuration

### Blast Radius
- 56 `any` occurrences across 15 files
- 17 files with Italian strings (code-level)
- Route changes affect: Sidebar.tsx, layout.tsx, all page.tsx files, any `<Link>` or `router.push` calls

</code_context>

<specifics>
## Specific Ideas

- The user explicitly wants user-facing Italian preserved — this is NOT an i18n project
- SSE_CONTRACT.md from Phase 2 is the source of truth for typing SSE events
- The evaluation folder rename (`valutazione` → `evaluation`) is the most user-visible change

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 03-frontend*
*Context gathered: 2026-04-02*
