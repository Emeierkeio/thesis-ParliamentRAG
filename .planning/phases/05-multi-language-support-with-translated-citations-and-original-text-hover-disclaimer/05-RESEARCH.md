# Phase 5: Multi-language Support - Research

**Researched:** 2026-04-04
**Domain:** i18n infrastructure, citation translation, SSE contract extension
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Languages supported: Italian (default) + English
- Default language: Italian (the app is for the Italian parliament)
- Language detection: No auto-detection — user selects language explicitly
- i18n framework: Claude's discretion (next-intl, react-i18next, or similar)
- Translation files: JSON-based locale files (it.json, en.json)
- All UI text currently hardcoded in Italian (Phase 3 decision) — must be extracted to translation keys
- Translation service: OpenAI (already used for generation) — no new dependencies
- Translation timing: On-the-fly during response generation, NOT pre-calculated
- When user language is Italian: No translation, show original citations as-is
- When user language is English: Translate each citation inline during generation, keep original Italian available
- Translation includes: Citation text (chunk_text/quote_text), debate titles, speaker roles
- Translation does NOT include: Speaker names, party names, session numbers, dates
- Hover behavior: Tooltip showing original Italian text when hovering over a translated citation
- Tooltip trigger: Hover on desktop, tap on mobile
- Tooltip content: Original Italian text + small "Originale" label
- Tooltip style: Subtle, non-intrusive, same width as citation
- Dual-layer disclaimer system:
  1. Banner: Fixed banner above the response area when language ≠ Italian
  2. Per-citation icon: Small globe icon on each translated citation
- Banner dismissable: Yes, with "Don't show again" stored in localStorage
- Icon always visible: Cannot be dismissed

### Claude's Discretion
- i18n library choice and configuration
- Exact OpenAI prompt for citation translation
- Tooltip animation and positioning
- How to handle translation failures (fallback to original)
- Language selector UI component (dropdown, toggle, etc.)
- Which UI strings to extract first (prioritization)
- SSE event modifications needed for translated content (if any)

### Deferred Ideas (OUT OF SCOPE)
- Additional languages beyond English (French, German, Spanish) — future milestone
- Pre-calculated translation cache for common citations — optimization for later
- Translation quality scoring / confidence indicator — too complex for v1
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| ML-01 | i18n infrastructure with next-intl or equivalent — Italian (default) + English locale files | Library selected: next-intl 4.9.0; without-i18n-routing setup; NEXT_LOCALE cookie; NextIntlClientProvider wraps root layout |
| ML-02 | Extract all hardcoded Italian UI text to translation keys across all pages and components | Inventory complete: 32 non-UI components contain Italian strings; 149 occurrences; grouped by file below |
| ML-03 | On-the-fly citation translation via OpenAI when user language ≠ Italian | Backend injection point identified: after `_build_verified_citations` in chat.py, before `citation_details` event; `make_async_client()` from key_pool.py reusable; new `translated_text` field in citation dict |
| ML-04 | Tooltip hover on translated citations showing original Italian text | shadcn/ui Tooltip already present and used in CitationCard; `TooltipProvider` already in root layout; wrap citation text block in Tooltip with original Italian in TooltipContent |
| ML-05 | Dual-layer disclaimer: dismissable banner + permanent globe icon on translated citations | Banner: new component above ChatArea; localStorage key `translationBannerDismissed`; globe icon: Lucide `Globe` icon inline with citation text preview |
</phase_requirements>

---

## Summary

The project currently has all UI text hardcoded in Italian across 32 non-UI component files. Phase 5 adds two distinct layers of i18n work: (1) static UI text extraction using next-intl 4.9.0 without URL-based locale routing — controlled entirely through a cookie — and (2) dynamic citation translation via OpenAI injected into the existing SSE pipeline.

The backend injection point for citation translation is well-defined: after `_build_verified_citations` builds the final citation list in `chat.py`, and before `citation_details` is emitted, a translation pass can be run for each citation's `text` and `full_text` fields when the user locale is `en`. The `make_async_client()` from `key_pool.py` is directly reusable. The SSE contract can be extended by adding optional `translated_text` and `translated_full_text` fields to the `CitationDict` without breaking the frozen contract (the contract prohibits removing/renaming fields, not adding new optional ones).

On the frontend, shadcn/ui's `Tooltip` component (using Radix UI primitives) is already installed and used extensively in `CitationCard.tsx`, `MessageBubble.tsx`, and `ProgressIndicator.tsx`. `TooltipProvider` is already mounted in `layout.tsx`. The citation tooltip wrapping is a localized change inside `CitationCard.tsx` with no structural rewrites needed.

**Primary recommendation:** Use next-intl 4.9.0 with `without-i18n-routing` setup (cookie-based via `NEXT_LOCALE`) and inject translation as a post-generation async pass in `chat.py` before emitting `citation_details`.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| next-intl | 4.9.0 | i18n for Next.js App Router | Official, maintained, native App Router support, no URL changes required, cookie-based locale via `without-i18n-routing` path |
| openai (Python) | already installed | Citation translation | Already used in pipeline; `make_async_client()` in `key_pool.py` handles key rotation |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @radix-ui/react-tooltip | already installed (^1.2.8) | Original Italian text tooltip | Already in project via shadcn/ui; `Tooltip`, `TooltipTrigger`, `TooltipContent` are the pattern in use |
| lucide-react | already installed (^0.563.0) | Globe icon for translated citations | `Globe` icon from existing package |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| next-intl | react-i18next | react-i18next has better ecosystem breadth but requires additional React context setup; next-intl has native App Router support with Server Component `getTranslations()` making it simpler for this architecture |
| next-intl | next-translate | next-translate is lighter but less actively maintained and lacks App Router-first documentation |

**Installation:**
```bash
cd frontend && npm install next-intl@4.9.0
```

**Version verification:** Confirmed 4.9.0 is the current latest (published 2026-04-02).

---

## Architecture Patterns

### Recommended Project Structure

```
frontend/
├── messages/
│   ├── it.json          # Italian (default) — all UI strings
│   └── en.json          # English translations
├── src/
│   ├── i18n/
│   │   └── request.ts   # getRequestConfig — reads NEXT_LOCALE cookie
│   ├── app/
│   │   └── layout.tsx   # Wrap children with NextIntlClientProvider
│   └── components/
│       └── layout/
│           └── LanguageSelector.tsx  # New: cookie-writing toggle
```

### Pattern 1: next-intl Without i18n Routing (Cookie-Based)

**What:** Locale determined by `NEXT_LOCALE` cookie, no URL changes, no `[locale]` directory wrapper needed in App Router.

**When to use:** App has no need for locale-specific URLs (confirmed by user decision: "no auto-detection").

**Setup (i18n/request.ts):**
```typescript
// Source: https://next-intl.dev/docs/getting-started/app-router/without-i18n-routing
import { getRequestConfig } from 'next-intl/server';
import { cookies } from 'next/headers';

export default getRequestConfig(async () => {
  const cookieStore = await cookies();
  const locale = cookieStore.get('NEXT_LOCALE')?.value ?? 'it';
  const validLocales = ['it', 'en'];
  const resolved = validLocales.includes(locale) ? locale : 'it';

  return {
    locale: resolved,
    messages: (await import(`../../messages/${resolved}.json`)).default,
  };
});
```

**next.config.ts update:**
```typescript
// Source: https://next-intl.dev/docs/getting-started/app-router/without-i18n-routing
import createNextIntlPlugin from 'next-intl/plugin';
const withNextIntl = createNextIntlPlugin('./src/i18n/request.ts');
export default withNextIntl(nextConfig);
```

**Root layout wrap:**
```typescript
// layout.tsx — add after resolving locale
import { NextIntlClientProvider } from 'next-intl';
import { getMessages, getLocale } from 'next-intl/server';

// Inside RootLayout:
const locale = await getLocale();
const messages = await getMessages();
// Wrap children: <NextIntlClientProvider messages={messages} locale={locale}>
```

**Client component usage:**
```typescript
import { useTranslations } from 'next-intl';
const t = useTranslations('ChatArea');
// t('placeholder') → "Cerca un tema..." or "Search a topic..."
```

**Language switching (writes cookie, triggers reload):**
```typescript
// LanguageSelector.tsx
document.cookie = `NEXT_LOCALE=en; path=/; max-age=31536000`;
window.location.reload();
```

### Pattern 2: Citation Translation — Backend Injection

**What:** After `_build_verified_citations()` in `chat.py`, run an async OpenAI translation pass on each citation's `text` and `full_text` fields when the request carries `Accept-Language: en` (or equivalent header/body param).

**Where translation lives:** `backend/app/services/translation.py` — new standalone service.

**Injection point in chat.py (line ~549):**
```python
# Existing:
verified_citations = await _build_verified_citations(...)
await emit("citation_details", {"citations": verified_citations})

# Phase 5 adds (before emit):
if request_locale == "en":
    verified_citations = await translate_citations(verified_citations)
# translate_citations adds translated_text / translated_full_text keys
# original text remains in text / full_text (contract preserved)
```

**Translation service pattern:**
```python
# backend/app/services/translation.py
async def translate_citation_batch(
    citations: list[dict],
    target_lang: str = "en"
) -> list[dict]:
    """Translate citation text fields; add translated_* fields."""
    client = make_async_client()  # reuse key_pool.py
    tasks = [_translate_one(c, client) for c in citations]
    return await asyncio.gather(*tasks, return_exceptions=False)

async def _translate_one(citation: dict, client) -> dict:
    text = citation.get("text", "")
    full_text = citation.get("full_text", "")
    if not text:
        return citation  # nothing to translate
    prompt = _build_prompt(text, full_text)
    try:
        resp = await client.chat.completions.create(
            model="gpt-4o-mini",  # fast, cheap — sufficient for translation
            messages=[{"role": "user", "content": prompt}],
            temperature=0,
        )
        result = json.loads(resp.choices[0].message.content)
        return {**citation,
                "translated_text": result.get("text", ""),
                "translated_full_text": result.get("full_text", "")}
    except Exception:
        return citation  # fallback: return original, no translated_* keys
```

**OpenAI prompt for parliamentary Italian → English:**
```
Translate the following Italian parliamentary speech excerpts to English.
Preserve formal parliamentary register. Do not translate proper nouns
(speaker names, party names, place names, dates, session numbers).
Return ONLY valid JSON: {"text": "...", "full_text": "..."}

text: {text}
full_text: {full_text}
```

### Pattern 3: Language Propagation Frontend → Backend

**What:** The frontend sends the user's current locale in a request header so the backend knows whether to translate.

**How:** Add `Accept-Language` header to the fetch call in `use-chat.ts`:
```typescript
const response = await fetch(`${config.api.baseUrl}/chat`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Accept-Language": currentLocale,  // "it" or "en"
  },
  body: JSON.stringify({ query: content, task_id: taskId }),
});
```

The FastAPI router reads it:
```python
from fastapi import Request
accept_lang = request.headers.get("accept-language", "it")
request_locale = "en" if "en" in accept_lang else "it"
```

### Pattern 4: Translated Citation Tooltip in CitationCard

**What:** When `citation.translated_text` is present (user is in English), show the translated text as the main content and wrap it in a Tooltip that reveals the original Italian on hover.

**Where:** `CitationCard.tsx`, inside the existing card body where `citation.text` is rendered (line ~116-130 of CitationCard.tsx):

```tsx
// In CitationCard — citation text preview block:
const displayText = citation.translated_text ?? citation.text ?? citation.quote_text ?? "";
const originalText = citation.translated_text ? (citation.text ?? citation.quote_text ?? "") : null;

{originalText ? (
  <Tooltip delayDuration={300}>
    <TooltipTrigger asChild>
      <p className="text-sm text-muted-foreground line-clamp-2 mb-3 leading-relaxed break-words cursor-help">
        &ldquo;{displayText}&rdquo;
        <Globe className="inline h-3 w-3 ml-1 text-muted-foreground/50" />
      </p>
    </TooltipTrigger>
    <TooltipContent side="bottom" className="max-w-[400px]">
      <p className="text-[10px] font-semibold text-muted-foreground/70 mb-1 uppercase tracking-wider">Originale</p>
      <p className="text-xs leading-relaxed">{originalText}</p>
    </TooltipContent>
  </Tooltip>
) : (
  <p className="text-sm text-muted-foreground line-clamp-2 mb-3 leading-relaxed break-words">
    &ldquo;{displayText}&rdquo;
  </p>
)}
```

The same pattern applies to the `CitationModal` full text view — `translated_full_text` / `full_text` swap.

### Anti-Patterns to Avoid

- **URL-based locale routing:** Do not create `src/app/[locale]/` directory — this project uses cookie-based locale without URL changes.
- **Translating LLM-generated prose:** The `chunk` SSE events carry the AI-generated answer text in Italian (the LLM writes in Italian, that's intentional). Only citations are translated, not the answer prose.
- **Translating speaker names and party names:** These are proper nouns and must never be translated. The prompt must explicitly forbid it.
- **Blocking the SSE stream for translation:** Run citation translation as a single `asyncio.gather` batch — do not await each citation sequentially.
- **Modifying frozen SSE event names or removing existing fields:** Only ADD `translated_text` / `translated_full_text` as optional new fields to the CitationDict. The `text` and `full_text` fields must remain untouched.
- **Storing locale in React state only:** Locale must survive page reload — use the `NEXT_LOCALE` cookie, not in-memory state.

---

## Italian String Inventory (ML-02)

Total occurrences of hardcoded Italian UI strings across non-UI TSX/TS files: approximately 149 visible strings. The key files with the most strings to extract:

### High Priority (user-facing core flow)

| File | Italian Strings Present (examples) | Count estimate |
|------|------------------------------------|---------------|
| `frontend/src/config/index.ts` | `progressSteps` labels + descriptions, chat placeholder, welcome message, `whyDescription` fields | ~25 strings |
| `frontend/src/components/chat/MessageBubble.tsx` | "Tu", "Camera dei Deputati — XIX Legislatura", "Fonti selezionate per autorità", "Citazioni", "Bilanciamento", section tooltip texts, "Bussola Ideologica", "Analisi High Quality (Best-of-N)", "Motivazione della scelta", "Variante A/B", "🏆 Vincitore", "Link copiato negli appunti", "Condividi", "Maggioranza", "Opposizione", "Compasso ideologico" | ~25 strings |
| `frontend/src/components/chat/CitationCard.tsx` | "Vai all'intervento originale", "Contesto Parlamentare", "Intervento", "Vai all'intervento sul sito della Camera" | ~6 strings |
| `frontend/src/components/shared/ProgressIndicator.tsx` | All 8 step short labels, all 8 step descriptions, "In corso...", "Tu", "Connessione...", waiting message templates | ~20 strings |
| `frontend/src/components/layout/Sidebar.tsx` | "Ricerca Topic", "Strumenti", "Ricerca Atti", "Analisi Autorità", "Compasso Ideologico", "Dati aggiornati al", "Impostazioni", "Documentazione", "Espandi menu", "Università degli Studi di Milano Bicocca" | ~12 strings |
| `frontend/src/hooks/use-chat.ts` | "Connessione...", "In attesa...", "Attualmente troppi utenti...", Italian progress labels, Italian error messages | ~10 strings |

### Medium Priority

| File | Italian Strings Present | Count estimate |
|------|------------------------|---------------|
| `frontend/src/components/chat/ExpertCard.tsx` | Expert display labels | ~5 strings |
| `frontend/src/components/chat/CompassCard.tsx` | "Significato:", axis labels | ~5 strings |
| `frontend/src/components/chat/TopicStatsModal.tsx` | "Dettaglio Statistiche" | ~5 strings |
| `frontend/src/components/shared/HistoryModal.tsx` | "Cronologia Chat" | ~5 strings |
| `frontend/src/components/settings/SettingsModal.tsx` | "Errore", "Salvato", "Impostazioni applicate correttamente.", "Editor Grafico", "Editor JSON" | ~6 strings |
| `frontend/src/app/rankings/page.tsx` | "Deputato", "Gruppo", "Coalizione" | ~3 strings |
| `frontend/src/app/explore/page.tsx` | "Errore Query" | ~2 strings |
| `frontend/src/app/layout.tsx` | metadata.description (IT), maintenance page strings, `lang="it"` | ~8 strings |

### Lower Priority (admin/internal)

| File | Italian Strings Present | Count estimate |
|------|------------------------|---------------|
| `frontend/src/components/settings/GraphicalEditors.tsx` | Settings UI descriptions | ~15 strings |
| `frontend/src/components/survey/SurveyModal.tsx` | Survey UI strings | ~10 strings |
| `frontend/src/components/search/*.tsx` | "Nessun gruppo trovato." | ~5 strings |

**Total extraction effort:** ~150 strings across ~15 files. The `config/index.ts` file requires special treatment because `progressSteps` is currently typed `as const` — the step labels must become translation keys rather than inline strings.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| i18n message interpolation, pluralization, date formatting | Custom string template system | next-intl `useTranslations`, ICU message format | Edge cases in plurals, number formatting, nested namespaces |
| Cookie locale detection | Custom middleware or cookie reader | next-intl's `NEXT_LOCALE` cookie convention | Already baked into next-intl's without-i18n-routing path |
| TypeScript type safety for translation keys | Manual key union types | next-intl `AppConfig` type augmentation with `messages` | Auto-generates types from JSON message files |
| Tooltip for original text | Custom hover div/popover | shadcn/ui `Tooltip` (already in project) | Already styled, accessible, mobile-aware via Radix |

---

## Common Pitfalls

### Pitfall 1: Translating Inside the SSE Stream (Blocking)

**What goes wrong:** Running `await _translate_one(citation)` sequentially for each citation adds N sequential OpenAI calls to the SSE latency before `citation_details` emits.

**Why it happens:** Naive sequential await inside a for-loop.

**How to avoid:** Use `asyncio.gather(*[_translate_one(c, client) for c in citations])` — all citations translate in parallel.

**Warning signs:** `citation_details` event arrives 10-15 seconds after `chunk` stream ends.

### Pitfall 2: next-intl and next.config.ts `output: 'standalone'`

**What goes wrong:** The `createNextIntlPlugin` wraps the Next.js config. If the current `nextConfig` already has Turbopack `resolveAlias`, the plugin must wrap the complete config including those aliases.

**Why it happens:** Plugin wrapping order matters — `withNextIntl(nextConfig)` must receive the full config object.

**How to avoid:**
```typescript
const withNextIntl = createNextIntlPlugin('./src/i18n/request.ts');
export default withNextIntl(nextConfig);  // nextConfig already includes turbopack config
```

**Warning signs:** Build fails with "Module not found: Can't resolve 'tailwindcss'" after adding plugin.

### Pitfall 3: Server Component vs Client Component Translation

**What goes wrong:** Using `useTranslations()` in a Server Component (or `getTranslations()` in a Client Component).

**Why it happens:** next-intl has two separate APIs: `useTranslations` (Client) and `getTranslations` (Server, async). Most of the target components are `"use client"` — `useTranslations` applies. `layout.tsx` is a Server Component — `getTranslations` applies.

**How to avoid:** Check `"use client"` at top of file. If present: `useTranslations()`. If absent: `await getTranslations()`.

**Warning signs:** TypeError at runtime about hooks being called in wrong context.

### Pitfall 4: config/index.ts `as const` Breaks Translation Keys

**What goes wrong:** `progressSteps` is typed `as const` — the `label` and `description` fields are literal types. If you replace them with translation keys, the `as const` type changes and any code that checks `step.label === "Analisi query"` breaks.

**Why it happens:** The config is used as a static definition AND as UI strings.

**How to avoid:** Keep `config/index.ts` using IDs/keys only. Create a separate translation namespace `progressSteps` in the JSON files. Components call `t('progressSteps.step1.label')` instead of reading from config. The `config.ui.progressSteps` keeps `id` and `icon` but drops `label`/`description`/`whyDescription` (these move to messages JSON).

**Warning signs:** TypeScript errors on `step.label` comparisons after changing config.

### Pitfall 5: Language Not Propagated to Backend at Query Time

**What goes wrong:** The language selector updates localStorage or the cookie client-side, but the `fetch` call in `use-chat.ts` doesn't include the locale — the backend translates nothing.

**Why it happens:** The cookie change happens in `LanguageSelector.tsx` but `use-chat.ts` doesn't read the cookie at query time.

**How to avoid:** Read `document.cookie` or a React context value at `sendMessage` call time and inject into the `Accept-Language` header. A `useLocale()` hook from next-intl client side provides this cleanly.

**Warning signs:** English locale selected, but citations arrive untranslated.

### Pitfall 6: Banner Shown to Italian Users

**What goes wrong:** The translation disclaimer banner renders for all users, including Italian-speaking users who see no translated content.

**How to avoid:** Render banner only when `locale !== 'it'` AND only when the current response has citations with `translated_text` present.

### Pitfall 7: Tooltip in CitationModal Full-Text View

**What goes wrong:** The citation modal (`CitationModal` inside `CitationCard.tsx`) shows `citation.full_text` — this must also show `translated_full_text` when available, with a hover to see the original. Easy to forget since the card and modal are separate render paths.

**How to avoid:** Create a single `useCitationText(citation)` utility hook that returns `{displayText, originalText, isTranslated}` and use it in both the card preview and the modal detail view.

---

## Code Examples

### next-intl Without i18n Routing Setup

```typescript
// Source: https://next-intl.dev/docs/getting-started/app-router/without-i18n-routing

// messages/it.json
{
  "ChatArea": {
    "placeholder": "Cerca un tema...",
    "historyTooltip": "Cronologia"
  },
  "MessageBubble": {
    "you": "Tu",
    "chamber": "Camera dei Deputati — XIX Legislatura",
    "citations": "Citazioni",
    "experts": "Fonti selezionate per autorità",
    "balance": "Bilanciamento",
    "compass": "Bussola Ideologica"
  }
}

// messages/en.json
{
  "ChatArea": {
    "placeholder": "Search a topic...",
    "historyTooltip": "History"
  },
  "MessageBubble": {
    "you": "You",
    "chamber": "Chamber of Deputies — XIX Legislature",
    "citations": "Citations",
    "experts": "Sources selected by authority",
    "balance": "Balance",
    "compass": "Ideological Compass"
  }
}
```

### Language Selector Component

```typescript
// components/layout/LanguageSelector.tsx
"use client";
import { useLocale } from 'next-intl';
import { Globe } from 'lucide-react';

export function LanguageSelector() {
  const locale = useLocale();

  const switchLocale = (newLocale: string) => {
    document.cookie = `NEXT_LOCALE=${newLocale}; path=/; max-age=31536000; SameSite=Lax`;
    window.location.reload();
  };

  return (
    <button
      onClick={() => switchLocale(locale === 'it' ? 'en' : 'it')}
      className="..."
      title={locale === 'it' ? 'Switch to English' : 'Passa all\'italiano'}
    >
      <Globe className="h-4 w-4" />
      <span>{locale.toUpperCase()}</span>
    </button>
  );
}
```

### Translation Banner Component

```typescript
// components/shared/TranslationBanner.tsx
"use client";
import { useState, useEffect } from 'react';
import { useTranslations, useLocale } from 'next-intl';
import { X } from 'lucide-react';

const DISMISS_KEY = 'translationBannerDismissed';

export function TranslationBanner({ hasCitations }: { hasCitations: boolean }) {
  const locale = useLocale();
  const t = useTranslations('TranslationBanner');
  const [dismissed, setDismissed] = useState(true);

  useEffect(() => {
    setDismissed(localStorage.getItem(DISMISS_KEY) === 'true');
  }, []);

  if (locale === 'it' || !hasCitations || dismissed) return null;

  const handleDismiss = () => {
    localStorage.setItem(DISMISS_KEY, 'true');
    setDismissed(true);
  };

  return (
    <div className="...">
      <p>{t('message')}</p>
      <button onClick={handleDismiss}>{t('dontShowAgain')}</button>
      <button onClick={() => setDismissed(true)}><X /></button>
    </div>
  );
}
```

---

## SSE Contract Impact

The SSE contract is **frozen** (documented in `backend/docs/SSE_CONTRACT.md`). The contract explicitly states:

> "Adding unexpected fields is safe; removing or renaming fields is breaking."

**Safe additions to `CitationDict`:**
- `translated_text: str | None` — translated preview text (when user locale is `en`)
- `translated_full_text: str | None` — translated full speech text (when user locale is `en`)
- `is_translated: bool` — flag indicating translation was applied

These fields are emitted inside the existing `citation_details` event (event #16 in chat.py, event #11 in query.py). No new event types are needed. The frontend `CitationCard` reads these as optional fields — when absent (Italian user), behavior is unchanged.

**No SSE event rename or reorder.** The `citation_details` event is the correct injection point because it carries the verified, deduplicated, final citation list — the data the user actually sees.

**query.py pipeline:** The query.py pipeline (`process_query_streaming`) also emits `citation_details`. It must receive the same translation injection. Both pipelines read `request_locale` from the `Accept-Language` header.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| next-intl with `[locale]` folder routing | `without-i18n-routing` (cookie, no URL change) | next-intl v3+ | Simpler for apps that don't need SEO per-locale |
| Global `AppProps` type augmentation for messages | Single `AppConfig` interface | next-intl v4.0 | Cleaner TypeScript, less global scope pollution |
| Separate `messages` + `formats` props required on `NextIntlClientProvider` | Auto-inherited from parent `NextIntlClientProvider` | next-intl v4.0 | Simpler layout.tsx |

**Deprecated/outdated:**
- `withLocale` from next-intl v3: replaced by `getRequestConfig` in v4
- Locale in URL as only mechanism: `localePrefix: 'never'` + cookie is the current pattern for apps without locale in URL

---

## Open Questions

1. **Translation latency budget**
   - What we know: Each OpenAI call (gpt-4o-mini) for a short citation (~200 chars) takes 300-800ms. With asyncio.gather, N citations translate in parallel limited by rate limits.
   - What's unclear: If a response has 15+ citations, parallel translation could still add 1-2 seconds before `citation_details` emits.
   - Recommendation: Set a timeout (e.g., 5s) on the translation batch. If it times out, fall back to untranslated citations.

2. **query.py locale propagation**
   - What we know: The `query.py` router (`process_query_streaming`) is called from a different endpoint than `chat.py`. Both need to receive the locale.
   - What's unclear: The query.py router signature — does it already receive the `Request` object from FastAPI?
   - Recommendation: Planner should verify the query.py router signature and ensure `request: Request` is available for header reading.

3. **Translated prose in the LLM answer**
   - What we know: User decision says only citations are translated, not the answer prose. The LLM generates in Italian.
   - What's unclear: Whether English users will find an Italian-prose answer (with English citations) jarring.
   - Recommendation: Out of scope per user decision; flag for phase 6 if needed.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | pytest (backend), no frontend test framework detected |
| Config file | `backend/tests/` existing structure |
| Quick run command | `cd backend && python -m pytest tests/unit/test_translation_service.py -x -q` |
| Full suite command | `cd backend && python -m pytest tests/unit/ -q` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| ML-01 | `request.ts` returns correct locale from `NEXT_LOCALE` cookie | unit | N/A (frontend, no test framework) | ❌ Wave 0 (no frontend tests) |
| ML-02 | All Italian UI strings have translation keys in it.json + en.json | unit | `python -m pytest tests/unit/test_translation_keys.py -x` | ❌ Wave 0 |
| ML-03 | `translate_citations()` adds `translated_text`; handles failures gracefully | unit | `python -m pytest tests/unit/test_translation_service.py -x` | ❌ Wave 0 |
| ML-03 | Backend chat.py emits `citation_details` with `translated_text` when `Accept-Language: en` | unit (source inspection) | `python -m pytest tests/unit/test_sse_contract.py -x` | ✅ (extend existing) |
| ML-04 | CitationCard renders Globe icon when `translated_text` present | manual smoke | N/A | manual only |
| ML-05 | `translationBannerDismissed` localStorage key hides banner | manual smoke | N/A | manual only |

### Sampling Rate
- Per task commit: `cd backend && python -m pytest tests/unit/test_translation_service.py -x -q`
- Per wave merge: `cd backend && python -m pytest tests/unit/ -q`
- Phase gate: Full backend suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `backend/tests/unit/test_translation_service.py` — covers ML-03 translation logic, batch gather, failure fallback
- [ ] `backend/tests/unit/test_translation_keys.py` — covers ML-02, verifies it.json and en.json have identical key sets
- [ ] `backend/app/services/translation.py` — the service itself (create before tests)
- Frontend: No test framework in place. ML-01, ML-04, ML-05 rely on manual visual smoke testing.

---

## Sources

### Primary (HIGH confidence)
- [next-intl without-i18n-routing docs](https://next-intl.dev/docs/getting-started/app-router/without-i18n-routing) — setup steps, cookie-based locale, `getRequestConfig`
- [next-intl v4.0 blog post](https://next-intl.dev/blog/next-intl-4-0) — breaking changes from v3, TypeScript improvements, GDPR cookie behaviour
- `backend/docs/SSE_CONTRACT.md` (project file) — frozen contract, addition-safe rule
- `frontend/src/components/ui/tooltip.tsx` (project file) — Tooltip component already installed
- `frontend/package.json` (project file) — confirmed no i18n library present, next@16.1.4, react@19.2.3

### Secondary (MEDIUM confidence)
- npm registry: `npm view next-intl version` returned 4.9.0 (verified 2026-04-04)
- [next-intl GitHub issue #1334](https://github.com/amannn/next-intl/issues/1334) — community confirmation of cookie-based locale in App Router without URL change

### Tertiary (LOW confidence)
- Italian string count (149) — estimated via grep pattern matching; exact count requires manual audit of all files

---

## Metadata

**Confidence breakdown:**
- Standard stack (next-intl 4.9.0): HIGH — npm registry confirmed, docs verified
- i18n without-i18n-routing pattern: HIGH — official docs read directly
- Citation translation injection point: HIGH — source code read directly, `_build_verified_citations` location confirmed
- Tooltip implementation: HIGH — `tooltip.tsx` read directly, pattern already used in CitationCard
- Italian string inventory: MEDIUM — grep-based estimate; exact count/grouping needs manual pass during implementation
- Translation latency: LOW — no production benchmarks available; 300-800ms per call is training data estimate

**Research date:** 2026-04-04
**Valid until:** 2026-05-04 (next-intl moves frequently; verify version before install)
