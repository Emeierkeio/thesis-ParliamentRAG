---
phase: 05-multi-language-support-with-translated-citations-and-original-text-hover-disclaimer
plan: 01
subsystem: ui
tags: [next-intl, i18n, translation, openai, asyncio, cookies]

# Dependency graph
requires: []
provides:
  - next-intl 4.9.0 installed with cookie-based locale (no URL routing)
  - src/i18n/request.ts reading NEXT_LOCALE cookie for locale resolution
  - NextIntlClientProvider wrapping root layout
  - Skeleton it.json/en.json with matching namespaces (Common, LanguageSelector, TranslationBanner, CitationCard)
  - LanguageSelector component writing NEXT_LOCALE cookie and reloading
  - backend translate_citation_batch with parallel asyncio.gather and error fallback
affects:
  - 05-02 (citation hover tooltip)
  - 05-03 (translation banner)
  - All components using useTranslations()

# Tech tracking
tech-stack:
  added: [next-intl@4.9.0]
  patterns:
    - Cookie-based locale switching (no [locale] URL segment routing)
    - next-intl server API (getLocale, getMessages) in async Server Component layout
    - OpenAI parallel batch translation via asyncio.gather with return_exceptions=True
    - Translation fallback: citation returned unchanged on any exception, no crash

key-files:
  created:
    - frontend/src/i18n/request.ts
    - frontend/messages/it.json
    - frontend/messages/en.json
    - frontend/src/components/layout/LanguageSelector.tsx
    - backend/app/services/translation.py
    - backend/tests/unit/test_translation_service.py
  modified:
    - frontend/next.config.ts
    - frontend/src/app/layout.tsx
    - frontend/package.json

key-decisions:
  - "Cookie-based locale (NEXT_LOCALE) selected over [locale] URL routing to avoid mass route migration"
  - "next-intl 4.9.0 pinned explicitly in npm install"
  - "translate_citation_batch returns citations unchanged (no translated_* keys) on failure, not partial dict"
  - "gpt-4o-mini chosen for translation model — cost-efficient for citation batch workloads"
  - "TRANSLATION_PROMPT instructs to not translate proper nouns (speaker names, party names, dates, session numbers)"

patterns-established:
  - "Pattern: Cookie locale via NEXT_LOCALE, read in request.ts getRequestConfig, written by LanguageSelector on click + reload"
  - "Pattern: Backend translation fallback — return_exceptions=True in asyncio.gather, log warning, return original citation"

requirements-completed: [ML-01, ML-03]

# Metrics
duration: 18min
completed: 2026-04-04
---

# Phase 5 Plan 01: i18n Infrastructure and Backend Translation Service Summary

**next-intl 4.9.0 with cookie-based locale switching, skeleton locale files, and parallel asyncio OpenAI citation translation with graceful fallback**

## Performance

- **Duration:** ~18 min
- **Started:** 2026-04-04T00:00:00Z
- **Completed:** 2026-04-04T00:18:00Z
- **Tasks:** 2 (Task 1: frontend i18n setup, Task 2: backend translation service TDD)
- **Files modified:** 9

## Accomplishments
- next-intl 4.9.0 installed and configured with cookie-based locale (NEXT_LOCALE cookie, no [locale] URL routing)
- Root layout converted to async Server Component wrapping children in NextIntlClientProvider
- LanguageSelector client component toggles IT/EN by writing NEXT_LOCALE cookie and reloading
- Skeleton it.json/en.json created with matching namespaces for TranslationBanner, CitationCard, LanguageSelector, Common
- Backend translate_citation_batch uses asyncio.gather with return_exceptions=True for parallel OpenAI calls, falling back to original citation on any failure

## Task Commits

Each task was committed atomically:

1. **Task 1: Install next-intl and configure i18n infrastructure with language selector** - `0c9e63b` (feat)
2. **Task 2 RED: Failing tests for citation translation service** - `98deebb` (test)
3. **Task 2 GREEN: Implement citation translation service** - `f68d406` (feat)

_TDD task has two commits (test RED then implementation GREEN)_

## Files Created/Modified
- `frontend/src/i18n/request.ts` - getRequestConfig reading NEXT_LOCALE cookie, dynamic locale message import
- `frontend/messages/it.json` - Italian locale skeleton with 4 namespaces
- `frontend/messages/en.json` - English locale skeleton with matching 4 namespaces
- `frontend/src/components/layout/LanguageSelector.tsx` - Globe icon toggle writing NEXT_LOCALE cookie
- `frontend/next.config.ts` - Wrapped with createNextIntlPlugin pointing to src/i18n/request.ts
- `frontend/src/app/layout.tsx` - Converted to async, added NextIntlClientProvider around TooltipProvider
- `frontend/package.json` - next-intl@^4.9.0 added to dependencies
- `backend/app/services/translation.py` - TRANSLATION_PROMPT constant, translate_citation_batch, _translate_one
- `backend/tests/unit/test_translation_service.py` - 7 unit tests covering all TDD behavior specs

## Decisions Made
- Cookie-based locale over [locale] URL segment routing: avoids renaming every route directory and rewriting all Link hrefs
- next-intl version 4.9.0 explicitly pinned to match plan spec
- translate_citation_batch returns the original citation dict without any `translated_*` keys on failure (not a partial dict with empty strings) — cleaner fallback behavior
- TRANSLATION_PROMPT instructs model to not translate proper nouns to preserve parliamentary names and dates faithfully
- gpt-4o-mini for translation model (cost-efficient, adequate for citation text)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] Fixed test import paths from `backend.app.services` to `app.services`**
- **Found during:** Task 2 RED phase
- **Issue:** Initial test file used `from backend.app.services.translation import ...` which fails when running pytest from the backend/ directory (where `app` is the root package)
- **Fix:** Updated all imports in test file to use `from app.services.translation import ...` and `patch("app.services.translation.make_async_client", ...)`, matching the pattern in existing test files (e.g., test_experts.py)
- **Files modified:** backend/tests/unit/test_translation_service.py
- **Verification:** pytest ran successfully after fix
- **Committed in:** 98deebb (RED commit, tests now fail for correct reason: module not found)

---

**Total deviations:** 1 auto-fixed (Rule 3 - blocking import path)
**Impact on plan:** Trivial path correction. No scope change.

## Issues Encountered
- NumPy 2.x compatibility warning appears in test stderr (known pre-existing issue in this environment, documented in STATE.md). Does not affect test results for translation service tests which have no NumPy dependency.

## User Setup Required
None - no external service configuration required for this plan.

## Next Phase Readiness
- i18n infrastructure is ready for 05-02 (citation hover tooltips with translated text) and 05-03 (translation banner)
- Any component can call `useTranslations('namespace')` from next-intl after adding keys to it.json/en.json
- Backend translate_citation_batch ready for integration into query pipeline (05-02)
- No blockers.

---
*Phase: 05-multi-language-support*
*Completed: 2026-04-04*

## Self-Check: PASSED

All created files exist and all task commits are present in git history.
