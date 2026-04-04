# Phase 5: Multi-language Support - Context

**Gathered:** 2026-04-04
**Status:** Ready for planning

<domain>
## Phase Boundary

Add multi-language support to the entire application: UI available in Italian (default) and English, parliamentary citations translated on-the-fly via OpenAI during response generation, tooltip hover to see the original Italian text, and a dual-layer disclaimer system (banner + per-citation icon) communicating that translations are machine-generated.

</domain>

<decisions>
## Implementation Decisions

### Languages & UI Translation

- **Languages supported:** Italian (default) + English
- **Default language:** Italian (the app is for the Italian parliament)
- **Language detection:** No auto-detection — user selects language explicitly
- **i18n framework:** Claude's discretion (next-intl, react-i18next, or similar)
- **Translation files:** JSON-based locale files (it.json, en.json)
- **All UI text currently hardcoded in Italian** (Phase 3 decision) — must be extracted to translation keys

### Citation Translation

- **Translation service:** OpenAI (already used for generation) — no new dependencies
- **Translation timing:** On-the-fly during response generation, NOT pre-calculated
- **When user language is Italian:** No translation, show original citations as-is
- **When user language is English:** Translate each citation inline during generation, keep original Italian available
- **Translation includes:** Citation text (chunk_text/quote_text), debate titles, speaker roles
- **Translation does NOT include:** Speaker names, party names, session numbers, dates (these are proper nouns/data)

### Original Text Display

- **Hover behavior:** Tooltip showing original Italian text when hovering over a translated citation
- **Tooltip trigger:** Hover on desktop, tap on mobile
- **Tooltip content:** Original Italian text + small "Originale" label
- **Tooltip style:** Subtle, non-intrusive, same width as citation

### Disclaimer & Transparency

- **Dual-layer system:**
  1. **Banner:** Fixed banner above the response area when language ≠ Italian: "Le citazioni sono tradotte automaticamente dall'italiano. Passa il mouse per vedere l'originale." / "Citations are automatically translated from Italian. Hover to see the original."
  2. **Per-citation icon:** Small globe icon (🌐) on each translated citation indicating machine translation
- **Banner dismissable:** Yes, with "Don't show again" option stored in localStorage
- **Icon always visible:** Cannot be dismissed — permanent transparency indicator

### Claude's Discretion
- i18n library choice and configuration
- Exact OpenAI prompt for citation translation
- Tooltip animation and positioning
- How to handle translation failures (fallback to original)
- Language selector UI component (dropdown, toggle, etc.)
- Which UI strings to extract first (prioritization)
- SSE event modifications needed for translated content (if any)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Frontend (being extended)
- `frontend/src/app/layout.tsx` — Root layout where language provider would wrap the app
- `frontend/src/components/chat/MessageBubble.tsx` — Where citations are rendered
- `frontend/src/components/chat/CitationCard.tsx` — Citation display component
- `frontend/src/hooks/use-chat.ts` — SSE event handling (may need translation integration)
- `frontend/src/types/sse.ts` — SSE event types (frozen from Phase 2)

### Backend (may need changes)
- `backend/app/services/generation/pipeline.py` — Generation pipeline where translation could be injected
- `backend/app/services/generation/sectional.py` — Per-party section generation with citations
- `backend/docs/SSE_CONTRACT.md` — Frozen SSE event contract (translation must not break it)

### Prior decisions
- `.planning/phases/03-frontend/03-CONTEXT.md` — UI text stays Italian for Italian users (now becomes Italian as default locale)
- `.planning/phases/02-backend/02-CONTEXT.md` — SSE payload field names frozen as snake_case

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `frontend/src/config/index.ts` — App configuration (add language setting)
- `frontend/src/hooks/` — Custom hooks pattern (add useLocale or useTranslation hook)
- `backend/app/key_pool.py` — OpenAI key management (reuse for translation calls)

### Established Patterns
- Next.js App Router with file-based routing
- Tailwind CSS for styling (tooltip styling)
- shadcn/ui for UI components (Tooltip component available)
- SSE streaming for real-time data

### Integration Points
- Language selector in layout/header or settings
- Translation layer between generation pipeline and SSE emission
- Citation rendering components need tooltip wrapper
- localStorage for user preference persistence

</code_context>

<specifics>
## Specific Ideas

- When language is Italian, the app behaves exactly as it does now — zero changes to the existing experience
- Translation should feel seamless — not like a separate "translation mode"
- The globe icon should be small and unobtrusive but always present on translated citations

</specifics>

<deferred>
## Deferred Ideas

- Additional languages beyond English (French, German, Spanish) — future milestone
- Pre-calculated translation cache for common citations — optimization for later
- Translation quality scoring / confidence indicator — too complex for v1

</deferred>

---

*Phase: 05-multi-language-support*
*Context gathered: 2026-04-04*
