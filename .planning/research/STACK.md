# Technology Stack — ParliamentRAG Refactoring

**Project:** ParliamentRAG Full Codebase Refactoring
**Researched:** 2026-04-02
**Overall confidence:** HIGH (Neo4j naming, FastAPI patterns) / MEDIUM (chunking recommendations)

---

## What This Covers

This document answers "what patterns and conventions should every layer of the refactored codebase follow?" It does not recommend new technologies — the stack is fixed (Python 3.11+, Neo4j 5.x, FastAPI, Next.js). It answers the design questions within that stack.

---

## 1. Neo4j Schema Naming Conventions

**Source:** Official Neo4j Cypher Manual + Cypher Style Guide (HIGH confidence)

### Canonical Rules

| Element | Convention | Rationale | Example (current → refactored) |
|---------|-----------|-----------|--------------------------------|
| Node labels | PascalCase | Official recommendation. Case-sensitive — `:Session` ≠ `:session`. | `Seduta → Session`, `Intervento → Speech` (already done in production schema) |
| Relationship types | SCREAMING_SNAKE_CASE | Verb-phrase convention with underscore separators | `HAS_CHUNK`, `SPOKEN_BY`, `MEMBER_OF_GROUP` (current is correct) |
| Property keys | camelCase | Official Cypher style guide. Single word lowercase is also fine. | `start_date → startDate`, `surname_name → surnameNameFallback` |
| Index names | snake_case descriptive | Convention in Neo4j GraphRAG packages. Descriptive over terse. | `chunk_embedding_index` (current is correct) |
| Parameters in Cypher | camelCase prefixed `$` | Follows property key convention | `$speakerId`, `$queryEmbedding` |

### Current Schema: What Is Already Correct

The production English schema (in `build_and_update.py`) already follows Node Label conventions correctly:
- `Session`, `Debate`, `Phase`, `Speech`, `Chunk`, `Deputy`, `GovernmentMember`, `ParliamentaryGroup`, `Committee`, `ParliamentaryAct`, `Vote`
- All relationship types are already SCREAMING_SNAKE_CASE

### Current Schema: What Needs Normalization

**Properties with mixed or Italian naming — change to camelCase:**

| Node | Current property | Refactored property | Notes |
|------|-----------------|---------------------|-------|
| `Chunk` | `start_char_raw` | REMOVE | Never used by backend — delete entirely |
| `Chunk` | `end_char_raw` | REMOVE | Never used by backend — delete entirely |
| `Chunk` | `text` | `text` | Keep as-is (already English) |
| `Chunk` | `index` | `index` | Keep as-is |
| `Speech` | `preprocessed_text` | REMOVE | Rename raw `text` → `text`; drop preprocessed_text |
| `Speech` | `text` (raw) | `text` | Keep only preprocessed form under this name |
| `Speech` | `surname_name` | `speakerNameFallback` | Used only for orphan reconciliation; rename to clarify intent |
| `Session` | `complete_date` | REMOVE | Keep only Neo4j Date `date` property |
| `Session` | `date` | `date` | Keep (already correct type: Neo4j Date) |
| Any node | `id` | `id` | Acceptable as-is — single word, unambiguous |

**Decision rule:** Property keys are camelCase per the Cypher style guide. Single-word properties (`id`, `text`, `date`, `title`) are already fine — no change needed. Multi-word properties must use camelCase. Drop snake_case variants.

### Cypher Query Style Rules (for all .py files)

```cypher
-- CORRECT: keyword uppercase, properties camelCase, labels/rels correct case
MATCH (c:Chunk)<-[:HAS_CHUNK]-(s:Speech)-[:SPOKEN_BY]->(d:Deputy)
WHERE c.id = $chunkId
RETURN c.text AS chunkText, d.firstName AS firstName

-- WRONG: mixed case keywords, snake_case properties
match (c:chunk)<-[:has_chunk]-(s:speech)
where c.id = $chunk_id
return c.text
```

**Formatting rules:**
- Each Cypher clause starts on its own line
- Keywords UPPERCASE
- 80-character line limit where practical
- Always include node labels in MATCH patterns (never bare `(n)` without label in production queries)
- Always parameterize values — never string interpolation for node properties

---

## 2. Neo4j Python Driver Patterns

**Source:** Official Neo4j Python Driver Manual 5.x + Neo4j performance docs (HIGH confidence)

### Session Management: The Right Pattern

**Current code in `neo4j_client.py`** uses `session.run()` (auto-commit) wrapped in a context manager. This works but misses retry logic and is not the recommended pattern for reads.

**Recommended migration:**

```python
# CURRENT (auto-commit, no retry):
def query(self, cypher: str, parameters: dict) -> list[dict]:
    with self.session() as session:
        result = session.run(cypher, parameters or {})
        return [record.data() for record in result]

# RECOMMENDED (managed transactions with retry for reads):
def query(self, cypher: str, parameters: dict) -> list[dict]:
    with self._driver.session(database="neo4j") as session:
        return session.execute_read(
            lambda tx: [r.data() for r in tx.run(cypher, parameters or {})]
        )

# RECOMMENDED (managed transaction for writes):
def execute_write(self, cypher: str, parameters: dict) -> None:
    with self._driver.session(database="neo4j") as session:
        session.execute_write(
            lambda tx: tx.run(cypher, parameters or {})
        )
```

**Why:** `execute_read` and `execute_write` provide automatic retry on transient errors. The callback must be idempotent (safe to retry). Auto-commit `session.run()` is acceptable only for one-off operations where retry is undesirable (e.g., SSE streaming where partial results were already sent).

**Critical rule:** Never return the `Result` object from a transaction function. Always consume it inside the callback (convert to list, extract values, etc.). Returning `Result` and consuming it after `tx.run()` closes is a common bug.

### Database Name: Always Specify Explicitly

```python
# WRONG — forces server round-trip to discover home database:
session = self._driver.session()

# CORRECT — specify explicitly every time:
session = self._driver.session(database="neo4j")
```

The current `neo4j_client.py` correctly passes `database=database` with a default of `"neo4j"`. Keep this pattern across all sessions.

### Batch Ingestion: UNWIND Pattern

For the build pipeline, all multi-node ingestion must use `UNWIND` with parameterized batches.

```python
# WRONG — one transaction per item (N round-trips):
for chunk in chunks:
    session.run("CREATE (c:Chunk {id: $id, text: $text})", chunk)

# CORRECT — batch with UNWIND (one transaction, one round-trip):
def _ingest_chunks_batch(tx, batch: list[dict]) -> None:
    tx.run(
        """
        UNWIND $batch AS row
        MERGE (c:Chunk {id: row.id})
        SET c.text = row.text, c.index = row.index
        """,
        batch=batch
    )

with driver.session(database="neo4j") as session:
    for i in range(0, len(chunks), 5000):
        session.execute_write(_ingest_chunks_batch, chunks[i:i+5000])
```

**Batch size:** 1,000–5,000 nodes per transaction is the effective range. The build pipeline currently calls `MERGE` for each item separately — this is the primary performance bottleneck to fix.

### MERGE vs CREATE in Build Pipeline

```python
# Use MERGE for: nodes that may already exist (Deputy, ParliamentaryGroup,
#                Committee, Session — idempotent ingestion)
# Use CREATE for: nodes that are definitionally new per build
#                (Chunk — unique IDs generated from hash, never pre-existing)

# MERGE with only the identifying key, SET the rest:
MERGE (d:Deputy {id: $deputyId})
SET d.firstName = $firstName, d.lastName = $lastName, d.cameraProfileUrl = $cameraProfileUrl
```

**Why CREATE for Chunk over MERGE:** MERGE requires a MATCH before CREATE, doubling the index lookups. Chunk IDs are content-addressed hashes — if the DB is rebuilt from scratch (as required), they cannot pre-exist.

### Connection Pool Configuration

Current `neo4j_client.py` config:
- `max_connection_pool_size=50` — appropriate for a single-worker deployment
- `max_connection_lifetime=3600` — acceptable
- `connection_acquisition_timeout=60.0` — acceptable

For the build pipeline (which runs single-process, high-throughput ingestion), pool size can be reduced to `10`. The backend server can keep `50`.

**Add `liveness_check_timeout`** to catch stale connections from load balancers:
```python
GraphDatabase.driver(
    uri,
    auth=(user, password),
    max_connection_pool_size=50,
    liveness_check_timeout=30,  # ADD THIS
    max_connection_lifetime=3600,
    connection_acquisition_timeout=60.0,
)
```

---

## 3. FastAPI Service Architecture

**Source:** FastAPI official docs, zhanymkanov/fastapi-best-practices (HIGH confidence)

### Project Structure: Current vs Recommended

**Current layout (by file type):**
```
backend/app/
    routers/      # all 7 routers
    services/     # all services flat + subdirs
    models/
    config.py
```

**Current layout is acceptable for this codebase size.** The official recommendation to restructure by domain applies to large teams with distinct domains. ParliamentRAG has one domain (parliament RAG) and 7 routers that all share the same Neo4j client, authority scorer, and generation pipeline. Restructuring by domain would scatter related code without benefit.

**What to change:** Improve the existing layout without reorganizing by domain.

### Dependency Injection: Replace Global Dict with FastAPI DI

**Current pattern in `deps.py`:** A global `get_services()` function that returns a dict. Routers call `services = get_services()` at the top of every function and access services as dict keys.

**Problem:** No type safety (dict access returns `Any`), no testability (hard to mock), dict key typos are silent.

**Recommended pattern:**

```python
# backend/app/services/deps.py
from functools import lru_cache
from fastapi import Depends
from typing import Annotated

@lru_cache
def get_neo4j_client() -> Neo4jClient:
    settings = get_settings()
    return Neo4jClient(settings.neo4j_uri, settings.neo4j_user, settings.neo4j_password)

@lru_cache
def get_retrieval_engine(
    neo4j: Annotated[Neo4jClient, Depends(get_neo4j_client)]
) -> RetrievalEngine:
    return RetrievalEngine(neo4j)

# In router:
@router.post("/query")
async def query(
    request: QueryRequest,
    retrieval: Annotated[RetrievalEngine, Depends(get_retrieval_engine)],
    authority: Annotated[AuthorityScorer, Depends(get_authority_scorer)],
):
    ...
```

**Why:** Type-safe, mockable in tests, FastAPI caches within request scope automatically, no global state mutation.

**Caveat:** `lru_cache` on dependency functions gives process-lifetime singletons, equivalent to the current behavior. This is correct — Neo4j driver, retrieval engine, and generation pipeline are expensive to construct and must be singletons.

### Async/Sync Boundary: The Core Pattern

The Neo4j Python driver (non-async variant) is synchronous and blocking. The current code correctly uses `run_in_executor` to avoid blocking the FastAPI event loop. This is the right pattern and must be consistently applied.

```python
# CORRECT — offload blocking Neo4j call to threadpool:
result = await asyncio.get_running_loop().run_in_executor(
    None,
    lambda: services["neo4j"].query(cypher, params)
)

# WRONG — blocks entire event loop during Neo4j I/O:
result = services["neo4j"].query(cypher, params)  # in async def endpoint
```

**Alternative — use the AsyncDriver:** The `neo4j` package provides `AsyncGraphDatabase.driver()` and `AsyncDriver` which work natively with `await`. This would eliminate all `run_in_executor` calls.

**Recommendation:** Do NOT migrate to AsyncDriver during this refactoring. The current synchronous driver with `run_in_executor` is correct and well-understood. Migration to AsyncDriver is a separate, non-trivial effort (all transaction callbacks must be `async def`). The refactoring is about code quality, not async model changes.

**Do fix:** Ensure every blocking Neo4j call inside `async def` route handlers uses `run_in_executor`. The current `query.py` correctly does this. Audit all other routers for violations.

### Pydantic: Use V2 Patterns

Current code uses Pydantic v2 (confirmed by `model_dump()` usage in `query.py`). Enforce v2 patterns throughout:

```python
# Use Field constraints for validation:
class QueryRequest(BaseModel):
    query: str = Field(..., min_length=3, max_length=1000)
    top_k: int = Field(default=100, ge=10, le=500)
    date_start: Optional[str] = Field(default=None, pattern=r"^\d{4}-\d{2}-\d{2}$")

# Use model_config instead of class Config:
class MyModel(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True, frozen=True)

# Use model_dump() not .dict() (v1 compat shim):
data = request.model_dump()

# Avoid: Optional[X] = None when you mean X | None = None (v2 style):
speaker_id: str | None = None  # preferred over Optional[str] = None
```

### Router Response Models: Always Declare

```python
# WRONG — response shape is invisible to OpenAPI, no validation:
@router.get("/experts/{speaker_id}")
async def get_expert(speaker_id: str):
    return {"id": speaker_id, "score": 0.8}

# CORRECT — explicit response model, OpenAPI docs, automatic validation:
@router.get("/experts/{speaker_id}", response_model=ExpertResponse)
async def get_expert(speaker_id: str) -> ExpertResponse:
    ...
```

### Error Handling: Use HTTPException Consistently

All routers must raise `HTTPException` with appropriate status codes, never return error dicts:

```python
# WRONG:
return {"error": "Speaker not found"}

# CORRECT:
raise HTTPException(status_code=404, detail="Speaker not found")
```

---

## 4. Python Code Quality Standards

**Source:** PEP 8, PEP 484, Python 3.11+ type system (HIGH confidence)

### Type Hints: Full Coverage Required

```python
# ALL function signatures must have complete type hints:
def compute_authority(
    speaker_id: str,
    query_embedding: list[float],
    top_k: int = 10,
) -> dict[str, float]:  # Use built-in generics (Python 3.9+), not typing.Dict

# Use | None syntax (Python 3.10+) over Optional:
def get_speaker(speaker_id: str) -> dict[str, Any] | None:
    ...

# Use TypedDict for structured dicts passed between functions:
class SpeakerRecord(TypedDict):
    speaker_id: str
    first_name: str
    last_name: str
    authority_score: float
```

**Why built-in generics:** Python 3.9+ supports `list[float]` instead of `typing.List[float]`. The project is Python 3.11+. Remove all `from typing import List, Dict, Tuple` imports where the built-in equivalent is available.

### Naming Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| Functions/methods | `snake_case` | `compute_authority_score` |
| Classes | `PascalCase` | `AuthorityScorer` |
| Constants | `UPPER_SNAKE_CASE` | `CHUNK_SIZE = 1200` |
| Private methods | `_single_underscore` | `_fetch_speaker_details` |
| Module-level private vars | `_underscore` | `_client: Neo4jClient \| None = None` |
| Type aliases | `PascalCase` | `EmbeddingVector = list[float]` |

**Current violation:** Many internal variables in `query.py` use inconsistent naming (e.g., `authority_all`, `authority_details` — both fine, but some functions mix camelCase from Neo4j record keys with snake_case local vars). Rule: Neo4j record key aliases in Cypher (`RETURN c.id AS chunkId`) must use camelCase to match the property naming convention; Python variables receiving these values use snake_case.

### Docstrings: English Only, Google Style

```python
def create_chunks(text: str, speech_id: str, chunk_size: int = 1200) -> list[ChunkRecord]:
    """Divide preprocessed speech text into overlapping sentence-boundary chunks.

    Sentences are split on terminal punctuation, respecting Italian abbreviations
    (On., Art., lett., etc.) to avoid false boundaries.

    Args:
        text: Preprocessed speech text (parentheticals already removed).
        speech_id: Parent speech identifier, used to build chunk IDs.
        chunk_size: Target character count per chunk.

    Returns:
        List of chunk records with id, text, and index fields.
    """
```

**No Italian comments or docstrings anywhere in the refactored codebase.** The current `ingest_stenografici.py` is entirely Italian — this is the primary target.

### Dead Code: Remove, Do Not Comment Out

- Remove `ingest_stenografici.py` Italian `save_to_neo4j()` method entirely
- Remove `start_char_raw` / `end_char_raw` computation in `create_chunks()`
- Remove `alignment_map` parameter and all alignment logic from chunking
- Remove `preprocessed_text` storage on Speech nodes
- Remove `complete_date` string property on Session nodes

Comment-out blocks are forbidden in the refactored codebase. If code is not used, delete it. Version control provides the history.

---

## 5. Parliamentary Text Chunking Strategy

**Source:** Weaviate chunking guide, Firecrawl 2025 chunking guide, NAACL 2025 findings (MEDIUM confidence — general RAG research applied to parliamentary domain)

### Current Approach Assessment

The current `create_chunks()` in `ingest_stenografici.py`:
- Character-based size limit (1200 chars) — correct order of magnitude
- Sentence-aware boundary detection — correct approach
- Overlap via sentence backtracking — correct approach
- `alignment_map` for char offset tracking — dead code (never used by backend)

**The chunking logic itself is sound. The bugs are in the alignment tracking (which is removed) and the `text.find()` sentence span detection (which silently drops sentences).**

### What to Fix in Chunking

**Bug: Silent sentence skip via `text.find()`**

```python
# CURRENT (buggy) — silent skip if sentence not found:
idx = text.find(s, cursor)
if idx == -1:
    continue  # SILENT DROP — sentence disappears from chunk

# CORRECT — always advance cursor, assert no sentence is lost:
idx = text.find(s, cursor)
if idx == -1:
    # This indicates a preprocessing inconsistency — log it, don't silently drop
    logger.warning(f"Sentence not found in text for speech {speech_id}: {s[:50]!r}")
    continue
sentence_spans.append({"start": idx, "end": idx + len(s)})
cursor = idx + len(s)
```

**Fix: Remove alignment_map parameter entirely.** The `create_chunks` function signature becomes:
```python
def create_chunks(text: str, speech_id: str) -> list[ChunkRecord]:
```

No `alignment_map`, no `raw_text`, no `start_char_raw`, no `end_char_raw`.

### Recommended Sentence Splitter

Replace the current ad-hoc regex split with a cleaner, testable function:

```python
# Italian parliamentary text abbreviations that should NOT trigger sentence boundaries:
_ABBREVS = frozenset({
    "on", "sen", "prof", "dott", "avv", "sig", "sigg",
    "lett", "cfr", "art", "comma", "n", "pag", "v", "b",
    "gen", "col", "tel", "fig", "tab",
})

_SENTENCE_END = re.compile(r'([.!?])\s+')

def split_sentences(text: str) -> list[str]:
    """Split Italian parliamentary text into sentences.

    Respects common Italian abbreviations to avoid false sentence boundaries.
    Returns a list of non-empty sentence strings.
    """
    ...
```

**Do NOT add spaCy or pySBD as a dependency.** The current regex approach handles Italian abbreviations adequately for parliamentary text. Adding an NLP library for sentence splitting introduces:
- A large dependency (spaCy model download ~100MB for Italian)
- Processing overhead per chunk
- A new failure mode (model not found)

The parliamentary text is well-structured (formal, written Italian) — rule-based splitting with abbreviation awareness is sufficient and matches the NAACL 2025 finding that fixed-size chunking with simple rules often matches or beats complex semantic chunking.

### Chunk Parameters: Keep Current Values

- **Target size:** 1,200 characters — appropriate for ada-002 (1536d) embeddings, balances context vs precision
- **Overlap:** 250 characters — ~20% of chunk size, within the 10-20% recommendation
- **Minimum speech length:** 100 characters — keep filter, prevents trivial chunk ingestion

These values have been validated against the actual corpus. Do not change without re-running evaluations.

---

## 6. Frontend (Next.js / TypeScript)

**Source:** Next.js 14+ docs, TypeScript strict mode guidelines (MEDIUM confidence — no deep dive done)

### Enforce Strict TypeScript

All components must compile under `strict: true` in `tsconfig.json`. Current likely violations:
- Any `any` types in SSE event handlers (dynamic JSON parsing)
- Missing return type annotations on utility functions
- Implicit `any` from untyped API response shapes

**Pattern for SSE events:**

```typescript
// Define discriminated union for all SSE event types:
type SSEEvent =
  | { type: "progress"; step: number; message: string }
  | { type: "experts"; data: Record<string, ExpertInfo> }
  | { type: "citations"; data: CitationInfo[] }
  | { type: "compass"; data: CompassData }
  | { type: "answer"; text: string }
  | { type: "done" }
  | { type: "error"; message: string };

// Type-safe parsing:
function parseSSEEvent(raw: string): SSEEvent | null {
  try {
    return JSON.parse(raw) as SSEEvent;
  } catch {
    return null;
  }
}
```

### Co-locate Types with Domain

```
frontend/src/
  types/
    api.ts       # API request/response types (matches backend Pydantic models)
    evidence.ts  # CitationInfo, ExpertInfo, etc.
    compass.ts   # CompassData types
  lib/
    api.ts       # fetch utilities, SSE client
```

Types that mirror backend Pydantic models must be in `types/api.ts` and kept in sync manually when the backend response model changes.

### No `any` Policy

Configure ESLint rule `@typescript-eslint/no-explicit-any: error`. Where a value truly has unknown shape (e.g., SSE JSON before type narrowing), use `unknown` and narrow explicitly.

---

## 7. What NOT to Do

These are anti-patterns observed in the current codebase that the refactoring must eliminate.

| Anti-Pattern | Location | Why Bad | What to Do Instead |
|-------------|----------|---------|-------------------|
| Italian-language code comments and docstrings | `ingest_stenografici.py`, `build_and_update.py` header | Non-English codebase is a maintenance hazard | English-only, everywhere |
| `session.run()` in auto-commit for all queries | `neo4j_client.py` | No retry on transient failure | `execute_read()` / `execute_write()` for managed transactions |
| Per-item Neo4j writes in build pipeline | `build_and_update.py` | N database round-trips where 1 suffices | `UNWIND $batch` with 1000-5000 items |
| `start_char_raw` / `end_char_raw` computation | `ingest_stenografici.py` `create_chunks()` | The alignment_map is buggy and these fields are never read by the backend | Remove entirely |
| `preprocessed_text` + `text` on Speech | Schema | Two representations of the same data, only one used | One field `text` = preprocessed form |
| `complete_date` string + `date` Neo4j Date | Session schema | Redundant, type inconsistency | Keep only `date` (Neo4j Date type) |
| Global service dict `get_services()` | `deps.py` | Dict access is untyped, untestable | FastAPI Depends() with typed functions |
| Snake_case Neo4j property keys | Any new properties | Violates Cypher style guide | camelCase for all multi-word properties |
| `from typing import List, Dict, Optional` | All Python files | Python 3.9+ has built-in generics | `list`, `dict`, `X \| None` |
| Italian mixed into English naming | Various files | `cognome_nome`, `testo_raw`, `seduta_id` in what is otherwise English code | English throughout |
| Comments explaining what code does, not why | Various | Noise when code is readable | Comment only non-obvious logic decisions |
| `GOVERNMENT_GROUPS` dict duplicated in 2 files | `initialize_db.py` + `build_and_update.py` | Single source of truth violated | Move to `app_config.py` or dedicated module |

---

## 8. Recommended Package Versions

All pinned, no changes to tech stack.

| Package | Current/Recommended | Notes |
|---------|-------------------|-------|
| `neo4j` (Python driver) | `5.x` latest stable | Keep synchronous driver. Do not migrate to AsyncDriver in this milestone. |
| `fastapi` | Latest `0.115.x` | No breaking changes expected |
| `pydantic` | `v2.x` | Already using v2 (confirmed by `model_dump()` usage) |
| `uvicorn` | `0.30.x+` | Single worker (`--workers 1`) enforced by pipeline semaphore design |
| `Next.js` | `14.x` App Router | Current, keep |
| `openai` | `1.x` | Keep text-embedding-ada-002 at 1536d |

---

## Sources

- [Neo4j Cypher Naming Rules](https://neo4j.com/docs/cypher-manual/current/syntax/naming/) — HIGH confidence
- [Neo4j Cypher Style Guide](https://neo4j.com/docs/cypher-manual/current/styleguide/) — HIGH confidence
- [Neo4j Python Driver — Transactions](https://neo4j.com/docs/python-manual/current/transactions/) — HIGH confidence
- [Neo4j Python Driver — Performance](https://neo4j.com/docs/python-manual/current/performance/) — HIGH confidence
- [Neo4j Python Driver — Concurrency](https://neo4j.com/docs/python-manual/current/concurrency/) — HIGH confidence
- [FastAPI Bigger Applications](https://fastapi.tiangolo.com/tutorial/bigger-applications/) — HIGH confidence
- [FastAPI Best Practices (zhanymkanov)](https://github.com/zhanymkanov/fastapi-best-practices) — MEDIUM confidence (community guide, well-maintained)
- [Weaviate Chunking Strategies for RAG](https://weaviate.io/blog/chunking-strategies-for-rag) — MEDIUM confidence
- [Firecrawl Best Chunking Strategies 2025](https://www.firecrawl.dev/blog/best-chunking-strategies-rag) — MEDIUM confidence
- [5 Tips for Fast Batched Updates in Neo4j](https://medium.com/neo4j/5-tips-tricks-for-fast-batched-updates-of-graph-structures-with-neo4j-and-cypher-73c7f693c8cc) — MEDIUM confidence
- [FastAPI run_in_threadpool vs run_in_executor](https://sentry.io/answers/fastapi-difference-between-run-in-executor-and-run-in-threadpool/) — MEDIUM confidence
