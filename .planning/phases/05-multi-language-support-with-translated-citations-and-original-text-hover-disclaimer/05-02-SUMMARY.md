---
phase: "05"
plan: "02"
subsystem: "frontend-i18n"
tags: [i18n, next-intl, locale, refactor, typescript]
dependency_graph:
  requires: ["05-01"]
  provides: ["complete-i18n-coverage"]
  affects: ["all frontend components"]
tech_stack:
  added: []
  patterns:
    - "useTranslations hook at component top level"
    - "getTranslations for server components"
    - "Dynamic key access: tPs(`step${id}.label` as Parameters<typeof tPs>[0])"
    - "Pass t as prop to sub-components that can't call hooks (StepResultDetails)"
    - "Config-only arrays: id + icon; labels in locale files"
key_files:
  created: []
  modified:
    - "frontend/messages/it.json"
    - "frontend/messages/en.json"
    - "frontend/src/config/index.ts"
    - "frontend/src/app/layout.tsx"
    - "frontend/src/components/chat/ChatInput.tsx"
    - "frontend/src/components/chat/CitationCard.tsx"
    - "frontend/src/components/chat/CompassCard.tsx"
    - "frontend/src/components/chat/ExpertCard.tsx"
    - "frontend/src/components/chat/MessageBubble.tsx"
    - "frontend/src/components/chat/TopicStatsModal.tsx"
    - "frontend/src/components/layout/Sidebar.tsx"
    - "frontend/src/components/shared/HistoryModal.tsx"
    - "frontend/src/components/shared/ProgressIndicator.tsx"
    - "frontend/src/hooks/use-chat.ts"
decisions:
  - "Remove label/description/whyDescription from config.progressSteps; keep only id and icon — locale files are the single source of truth for UI text"
  - "Pass tPi as a prop to StepResultDetails (sub-function) to avoid hooks-in-non-component violation"
  - "Convert SYSTEM_TOUR_FEATURES to SYSTEM_TOUR_FEATURES_CONFIG with titleKey/descKey references, resolved in ProgressFullPage"
  - "Use as Parameters<typeof tPs>[0] cast to allow dynamic key construction for step labels"
metrics:
  duration: "~30 min (across two sessions)"
  completed_date: "2026-04-04"
  tasks_completed: 2
  files_modified: 14
---

# Phase 05 Plan 02: Extract All Hardcoded Italian UI Strings to Locale Keys — Summary

**One-liner:** Full i18n coverage via next-intl useTranslations across 13 components — ~150 hardcoded Italian strings extracted to it.json/en.json, config refactored to remove all label fields, zero tsc errors.

## Tasks Completed

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | High-priority files (~90 strings): locale files, config, MessageBubble, ProgressIndicator, Sidebar, layout.tsx, use-chat.ts, ChatInput | 7419aa7 | 9 files |
| 2 | Remaining components (~60 strings): CitationCard, ExpertCard, CompassCard, TopicStatsModal, HistoryModal | 6a33e40 | 5 files |

## What Was Built

Both `messages/it.json` and `messages/en.json` were comprehensively populated with namespaces covering all 14 modified components. The `config/index.ts` progressSteps array was stripped of `label`, `description`, and `whyDescription` fields — only `id` and `icon` remain. All 13 component files now use `useTranslations` (client) or `getTranslations` (server async) instead of hardcoded Italian strings.

Key namespaces added/extended: Chat (28 keys), ProgressIndicator (31 keys including shortLabels), ProgressSteps (8 steps × 3 keys), Sidebar (14 keys), MessageBubble (20 keys), ExpertCard (22 keys), TopicStatsModal (16 keys), CitationCard (8 keys), CompassCard (9 keys), HistoryModal (10 keys).

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] ChatInput referenced removed config.ui.chat.placeholder**
- **Found during:** Task 1 (tsc run)
- **Issue:** `config.ui.chat.placeholder` was removed from config but still referenced in ChatInput.tsx line 71 as fallback
- **Fix:** Removed fallback — ChatInput already accepts `placeholder` as a required prop from its parent
- **Files modified:** `frontend/src/components/chat/ChatInput.tsx`
- **Commit:** 7419aa7 (included in Task 1 commit)

**2. [Rule 1 - Bug] balance and compass cases used removed stepConfig?.label / stepConfig?.description**
- **Found during:** Task 1 (code review during use-chat.ts update)
- **Issue:** After removing label/description from config.progressSteps, references to `stepConfig?.label` and `stepConfig?.description` in the balance/compass SSE handlers would produce TS errors
- **Fix:** Replaced with `prev.stepLabel` / `prev.stepDescription` pass-through (no advancement of the stepper label text in those cases — progression happens via `data.message` in the progress event)
- **Files modified:** `frontend/src/hooks/use-chat.ts`
- **Commit:** 7419aa7

## Self-Check: PASSED

- FOUND: frontend/messages/it.json
- FOUND: frontend/messages/en.json
- FOUND: frontend/src/config/index.ts
- FOUND commit: 7419aa7 (Task 1)
- FOUND commit: 6a33e40 (Task 2)
- tsc --noEmit: zero errors after both tasks
