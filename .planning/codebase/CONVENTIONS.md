# Coding Conventions

**Analysis Date:** 2026-04-01

## Naming Patterns

**Files:**
- Python: `snake_case.py` (e.g., `query.py`, `authority_scorer.py`, `evaluation.py`)
- TypeScript/React: `camelCase.ts` or `PascalCase.tsx` for components
  - Hooks: `use-chat.ts`, `use-*.ts` (kebab-case for consistency)
  - Components: `SurveyModal.tsx`, `ExpertCard.tsx`, `MessageBubble.tsx` (PascalCase)
  - Pages: `page.tsx`, `[id]`, `[[...path]]` (Next.js file-based routing)
  - Utils: `utils.ts`, `config/index.ts` (lowercase or camelCase)

**Functions:**
- Python: `snake_case` for all functions and methods (e.g., `_compute_experts()`, `_normalize_for_verbatim()`)
  - Private/internal: prefix with `_` (e.g., `_fetch_speaker_details()`, `_rate_limited_query()`)
  - Async functions: same convention but prefixed with `async` keyword
- TypeScript: `camelCase` for functions (e.g., `sendMessage()`, `addUserMessage()`)
  - Event handlers: prefix with `handle` or `on` (e.g., `handleClick`, `onError`)
  - Utils: simple names (e.g., `cn()`, `formatDate()`)

**Variables:**
- Python: `snake_case` (e.g., `authority_scores`, `evidence_list`, `speaker_ids`)
  - Constants: `UPPER_SNAKE_CASE` (e.g., `MAX_CONCURRENT_PIPELINES`, `ALL_PARTIES`)
  - Module-level private: `_name` (e.g., `_pipeline_semaphore`)
- TypeScript: `camelCase` (e.g., `isLoading`, `streamingContent`, `currentTaskIdRef`)
  - Constants: `UPPER_SNAKE_CASE` (e.g., `BACKEND_URL`) or `camelCase` when exported
  - Refs: suffix with `Ref` (e.g., `abortControllerRef`, `sendMessageRef`)

**Types:**
- Python Pydantic models: `PascalCase` (e.g., `QueryRequest`, `AuthorityScore`, `Expert`)
  - BaseModel subclasses: use Field with descriptions
  - Enum-like: uppercase set definitions (e.g., `KNOWN_PARTIES`, `PARTY_KEYWORDS`)
- TypeScript interfaces: `PascalCase` (e.g., `Message`, `Citation`, `Expert`, `UseChatOptions`)
  - Type unions: `PascalCase | PascalCase` (e.g., `StreamEventType`)
  - CSS classes: `kebab-case` (via Tailwind, `cn()` utility)

## Code Style

**Formatting:**
- Python: Black formatter implied (requirements.txt includes `black>=24.1.0`)
  - Line length: not explicitly configured, Black default ~88 chars
  - Indentation: 4 spaces
- TypeScript: ESLint v9 (see `eslint.config.mjs`)
  - Uses `eslint-config-next/core-web-vitals` and `eslint-config-next/typescript`
  - No explicit Prettier config found; Next.js defaults apply

**Linting:**
- Python: mypy type checking implied (requirements.txt includes `mypy>=1.8.0`)
  - isort for import ordering (requirements.txt includes `isort>=5.13.0`)
- TypeScript: ESLint with Next.js rules
  - No explicit pre-commit hooks detected
  - Run via: `npm run lint` (defined in package.json)

## Import Organization

**Order (Python):**
1. Standard library imports (`import json`, `import logging`, `from datetime import date`)
2. Third-party imports (`from fastapi import`, `from pydantic import`, `import numpy as np`)
3. Local application imports (`from ..services import`, `from ...config import`)

**Order (TypeScript):**
1. React/Next.js imports (`import React`, `import { useState }`)
2. Third-party/UI library imports (`import { Button }`, `from @radix-ui`, `from lucide-react`)
3. Local imports (`import type { Message }`, `from @/components`, `from @/types`)
4. Type-only imports: use `import type { ... }`

**Path Aliases:**
- TypeScript: `@/*` maps to `./src/*` (defined in `tsconfig.json`)
  - Example: `import { cn } from "@/lib/utils"` instead of `../../lib/utils`

## Error Handling

**Patterns:**

**Python (FastAPI):**
- Use `try/except` blocks with specific exception types
- Log errors with `logger.error()` at exception point
- Raise `HTTPException(status_code=..., detail=...)` for API errors
- Pattern observed in `query.py`:
  ```python
  try:
      result = await services["retrieval"].retrieve(...)
  except Exception as e:
      logger.error(f"Query processing error: {e}")
      yield f"data: {json.dumps({'type': 'error', 'message': str(e)})}\n\n"
  ```
- For streaming responses: yield error events rather than raising
- Non-fatal errors (e.g., compass computation): log and continue pipeline

**TypeScript (React):**
- Use `try/catch` in async functions
- Log errors with `console.error()` including context
- Handle AbortError separately (stream cancellation)
- Pattern observed in `use-chat.ts`:
  ```typescript
  try {
    const response = await fetch(...);
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
  } catch (error) {
    if ((error as Error).name === "AbortError") {
      // Handle abort
    } else {
      const errorMessage = error instanceof Error ? error.message : "Unknown";
      options.onError?.(error instanceof Error ? error : new Error(errorMessage));
    }
  }
  ```
- Use optional callbacks for error propagation: `options.onError?.(error)`
- Mobile resilience: silent retry logic on stream failure

## Logging

**Framework:**
- Python: standard `logging` module with module-level logger
  - `logger = logging.getLogger(__name__)`
  - Level configured by `LOG_LEVEL` env var (default "INFO")
- TypeScript: `console.error()`, `console.warn()`, `console.log()`

**Patterns:**

**Python:**
- Log at entry points and key decision points
- Include context in log messages: `logger.error(f"[QUERY] {message}: {details}", exc_info=True)`
- Use log prefixes for tracing: `[QUERY]`, `[COMPASS]`, `[PIPELINE]`
- Example: `logger.error(f"[COMPASS] Failed (pipeline continues): {_compass_err}", exc_info=True)`

**TypeScript:**
- Log in error handlers with context prefix
- Include raw data for debugging: `console.error(\`Failed to parse: $\{data.slice(0, 200)}\`)`
- Example: `console.error(\`[Pipeline:Buffer] Failed to parse buffered data:\`, e, \`raw: "$\{line.substring(0, 200)}"\`)`

## Comments

**When to Comment:**
- Complex logic requiring explanation (especially temporal coalition crossing, authority normalization)
- Non-obvious query patterns or database interactions
- Algorithm implementation details (e.g., percentile normalization, KDE bandwidth)
- Workarounds or known limitations

**JSDoc/TSDoc:**
- Python: docstrings for all public functions and classes
  - Format: triple-quoted strings, Google-style or Sphinx
  - Example:
    ```python
    def process_query_streaming(request: QueryRequest) -> AsyncGenerator[str, None]:
        """Process query with SSE streaming.

        Yields SSE events as the pipeline progresses.
        """
    ```
- TypeScript: JSDoc for exported functions and complex types
  - Example:
    ```typescript
    /**
     * Parse markdown content to extract bold names (**Name**) grouped by section.
     * For government members, resolves institutional role instead of party.
     */
    ```

## Function Design

**Size:**
- Python: functions up to ~80-100 lines are common (e.g., `_compute_experts()` ~88 lines, `process_query_streaming()` ~415 lines)
  - Longer pipelines split into logical stages yielding progress events
  - Helper functions extracted for specific tasks (e.g., `_fetch_speaker_details()`, `_batch_fetch_deputy_cards()`)
- TypeScript: hooks and components range 100-500+ lines (e.g., `SurveyModal.tsx` 2325 lines)
  - Hooks encapsulate complex state logic (e.g., `use-chat.ts` 768 lines handles message streaming and retry)
  - Larger components extract UI sections into local helper functions

**Parameters:**
- Python: use Pydantic `BaseModel` for complex request objects
  - Single request object preferred over multiple parameters
  - Optional parameters use `Optional[T]` with defaults
- TypeScript: interfaces for options/config objects
  - Example: `interface UseChatOptions { onError?: (error: Error) => void }`
  - Callbacks passed as optional function properties

**Return Values:**
- Python: explicit types in function signatures
  - Async generators for streaming: `AsyncGenerator[str, None]`
  - Dict for flexible returns: `Dict[str, Any]` with documentation of keys
- TypeScript: explicit return types (enforced by strict tsconfig)
  - Example: `useCallback((content: string) => Promise<void>, [...])`
  - Component returns JSX.Element (implicit)

## Module Design

**Exports:**
- Python: public API via `__all__` or direct function/class definition
  - Services registered via dependency injection (see `get_services()`)
  - Routers mounted on FastAPI app (see `router = APIRouter(prefix="/api", tags=[...])`)
- TypeScript: named exports for types, utils, and components
  - Barrel files use re-exports: `export * from "./chat"` (in `types/index.ts`)
  - Default exports used for pages: `export default function Home() { ... }`

**Barrel Files:**
- `frontend/src/types/index.ts`: re-exports all type definitions
  - Usage: `import type { Message } from "@/types"` pulls from barrel
  - Centralizes type imports across application

## Configuration

**Pattern:**
- Python: YAML-based config (`config/default.yaml`) + environment secrets (`.env`)
  - Settings class (`Settings`) loads env vars only (API keys, Neo4j credentials)
  - ConfigLoader caches YAML and provides property accessors
  - Example: `get_config().authority` returns dict of authority weights
- TypeScript: centralized `config/index.ts` with hardcoded defaults
  - Backend URL from env: `process.env.NEXT_PUBLIC_API_URL || "/api"`
  - UI config (step labels, placeholders, timeouts) in object literal

## Testing Infrastructure

**Setup (requirements.txt):**
- `pytest>=7.4.0` - test runner
- `pytest-asyncio>=0.23.0` - async test support
- `pytest-cov>=4.1.0` - coverage measurement
- **No test files found in project** - tests not yet implemented

**Frontend:**
- ESLint runs on save/commit
- No Jest or Vitest configuration detected
- **No test files in frontend/src** - tests not yet implemented

---

*Convention analysis: 2026-04-01*
