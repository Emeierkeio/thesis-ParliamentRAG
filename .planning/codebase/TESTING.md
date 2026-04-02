# Testing Patterns

**Analysis Date:** 2026-04-01

## Test Framework

**Runner:**
- Backend: pytest (v7.4.0+)
  - Config: `backend/.pytest_cache/` exists; no explicit `pytest.ini` or `setup.cfg` found
  - Default config applies (tests must match `test_*.py` or `*_test.py` pattern)
- Frontend: ESLint configured for linting
  - No Jest, Vitest, or other test runner configured
  - No test files found

**Assertion Library:**
- Backend: pytest's built-in assertion introspection
- Frontend: Not applicable (no test framework)

**Run Commands:**
```bash
# Backend (Python)
pytest                 # Run all tests (from backend/ directory)
pytest --cov         # Run with coverage measurement
pytest -v            # Verbose output

# Frontend (TypeScript)
npm run lint         # Run ESLint (code quality, not functional tests)
npm run build        # Compile TypeScript (catches type errors)
```

## Test File Organization

**Location:**
- Backend: co-located pattern expected (test files alongside source or in `tests/` directory)
  - Current status: **No test files found** in `backend/app/`
  - `.pytest_cache/` exists, indicating pytest has been run at some point
- Frontend: **No test framework configured**
  - No `__tests__` or `.test.ts` files in `frontend/src/`

**Naming:**
- Backend expected: `test_*.py` or `*_test.py` files
  - Example: `test_query.py` would test `routers/query.py`
  - Example: `test_authority_scorer.py` would test `services/authority/scorer.py`
- Frontend: not applicable

## Test Structure

**Suite Organization (Expected Pattern):**

For Python, following pytest conventions:
```python
# test_query.py
import pytest
from app.routers.query import process_query_streaming, QueryRequest

class TestQueryEndpoint:
    """Tests for query routing and streaming."""

    @pytest.mark.asyncio
    async def test_process_query_streaming_success(self):
        """Test successful query processing yields SSE events."""
        request = QueryRequest(query="Test query", top_k=100)
        # arrange/act/assert

    def test_query_request_validation(self):
        """Test QueryRequest enforces min/max lengths."""
        # Pydantic validation tests
        with pytest.raises(ValidationError):
            QueryRequest(query="ab")  # Too short (min_length=3)

class TestAuthorityScoring:
    """Tests for authority computation."""

    def test_percentile_normalization(self):
        """Test percentile-based authority normalization."""
        # Test that all scores remain in [0, 1]
```

**Patterns (Expected):**
- Setup: Use `@pytest.fixture` for common test data (speakers, queries, evidence)
  - Example fixture: `neo4j_mock` with sample deputy/group data
  - Example fixture: `sample_query_request` with valid QueryRequest
- Teardown: Automatic via pytest (context managers for Neo4j connections)
- Assertion: pytest's assert statements with descriptive messages
  - Example: `assert authority_score["score"] >= 0.0 and authority_score["score"] <= 1.0`

## Mocking

**Framework:** unittest.mock (Python standard library)
  - Used via `from unittest.mock import Mock, patch, MagicMock`
  - Pattern: `@patch('app.services.neo4j_client.Neo4jClient')`

**Expected Patterns:**

```python
# Mock Neo4j client for testing without database
@patch('app.services.neo4j_client.Neo4jClient')
def test_compute_experts_with_mock_neo4j(mock_neo4j):
    """Test expert computation without real Neo4j."""
    mock_client = mock_neo4j.return_value
    mock_client.fetch_speaker_profile.return_value = {
        "speaker_id": "d300001",
        "name": "Mario Rossi",
        "group": "FRATELLI D'ITALIA"
    }

    # Test _compute_experts with mocked client
    # Assert expected output format

# Mock embedding service
@patch('app.services.retrieval.embed_query')
async def test_query_with_mock_embeddings(mock_embed):
    """Test query with mocked embedding service."""
    mock_embed.return_value = [0.1, 0.2, ..., 0.8]  # 1536-dim vector

    # Test retrieval ranking without calling OpenAI
```

**What to Mock:**
- Database calls: Neo4jClient methods (fetching deputies, interventions, acts)
- External APIs: OpenAI embeddings, GPT-4o generation
- I/O operations: file system reads, HTTP requests to parliamentary data sources
- Time-dependent logic: `datetime.now()` for temporal testing

**What NOT to Mock:**
- Pydantic model validation (test real validation behavior)
- Business logic: authority scoring algorithms, percentile normalization
- Data structure transformations (test real output format)
- Streaming SSE event formatting (test actual event structure)

## Fixtures and Factories

**Test Data (Expected Pattern):**

```python
# conftest.py - shared fixtures
import pytest
from datetime import date

@pytest.fixture
def sample_speaker():
    """Create a sample speaker for testing."""
    return {
        "speaker_id": "d300001",
        "first_name": "Mario",
        "last_name": "Rossi",
        "group": "FRATELLI D'ITALIA",
        "coalition": "maggioranza",
        "role": "Deputy",
        "profession": "Engineer",
        "education": "Master's in Engineering",
    }

@pytest.fixture
def sample_authority_score():
    """Create a sample authority score."""
    return {
        "speaker_id": "d300001",
        "score": 0.75,
        "components": {
            "profession_relevance": 0.6,
            "education_relevance": 0.5,
            "committee_relevance": 0.8,
            "acts_score": 0.7,
            "interventions_score": 0.9,
            "role_score": 0.3,
        }
    }

@pytest.fixture
def sample_query_request():
    """Create a valid QueryRequest."""
    return QueryRequest(
        query="Qual è la posizione dei partiti sull'immigrazione?",
        top_k=100,
        stream=True
    )
```

**Location:**
- Python: `backend/tests/conftest.py` (pytest auto-discovery)
  - Or co-located: `backend/app/routers/test_query.py` may have local fixtures

## Coverage

**Requirements:**
- Not enforced by configuration
- pytest-cov available (requirements.txt includes `pytest-cov>=4.1.0`)
- **No test files currently present** to measure coverage

**View Coverage:**
```bash
cd backend
pytest --cov=app --cov-report=html
# Generates htmlcov/index.html with coverage breakdown
```

## Test Types

**Unit Tests (Expected):**
- Scope: Individual functions and methods
- Approach: Mock external dependencies (Neo4j, OpenAI)
- Examples:
  - `test_parse_neo4j_date()` - date parsing edge cases
  - `test_percentile_normalization()` - authority scoring algorithm
  - `test_query_request_validation()` - Pydantic field constraints

**Integration Tests (Expected):**
- Scope: Multiple components working together
- Approach: Use test Neo4j instance or mock multiple services
- Examples:
  - `test_query_to_response_flow()` - full pipeline from QueryRequest to streaming output
  - `test_authority_computation_with_real_coefficients()` - real weight application with test data
  - `test_citation_verification_pipeline()` - citation extraction through verification

**E2E Tests (Not Implemented):**
- Framework: Would require Cypress, Playwright, or Selenium
- No E2E test framework installed

## Common Patterns

**Async Testing:**

```python
# Backend uses asyncio for streaming and concurrent operations
# Test pattern for async functions:

@pytest.mark.asyncio
async def test_process_query_streaming():
    """Test async generator for query streaming."""
    request = QueryRequest(query="test", top_k=100)

    events = []
    async for event_str in process_query_streaming(request):
        # Parse SSE event format: "data: {json}\n\n"
        json_str = event_str.replace("data: ", "").strip()
        events.append(json.loads(json_str))

    # Assert events in expected order: progress → experts → compass → citations → complete
    assert events[0]["type"] == "progress"
    assert events[-1]["type"] == "complete"
```

**Error Testing:**

```python
# Python: test exception raising
def test_query_request_too_short():
    """Test QueryRequest validation for minimum length."""
    with pytest.raises(ValueError):  # or ValidationError from pydantic
        QueryRequest(query="ab")  # min_length=3

# Python: test error response in streaming
@pytest.mark.asyncio
async def test_query_error_yields_error_event():
    """Test that query errors are properly yielded as SSE events."""
    bad_request = QueryRequest(query="test")

    # Mock services to raise exception
    with patch('app.services.retrieval.retrieve') as mock_retrieve:
        mock_retrieve.side_effect = Exception("DB connection failed")

        events = []
        async for event_str in process_query_streaming(bad_request):
            events.append(json.loads(event_str.replace("data: ", "")))

        # Last event should be error
        assert events[-1]["type"] == "error"
        assert "DB connection failed" in events[-1]["message"]
```

**Temporal Testing (Authority with Coalition Crossing):**

```python
def test_authority_invalidated_on_coalition_crossing():
    """Test that authority is invalidated when deputy crosses coalitions."""
    # Setup: deputy was in opposition on date1, crossed to maggioranza on date2
    authority_date2 = compute_authority(
        speaker_id="d300001",
        query_embedding=[...],
        reference_date=date(2025, 3, 15)  # After coalition crossing
    )

    # Authority should be invalidated for pre-crossing acts/speeches
    assert authority_date2["authority_invalidated_periods"] != []
    assert authority_date2["score"] < 0.5  # Reduced due to invalidation
```

## Test Configuration

**pytest setup (if created):**

```ini
# backend/pytest.ini
[pytest]
asyncio_mode = auto
testpaths = tests
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
```

**conftest.py (if created):**

```python
# backend/conftest.py
import pytest
from app.config import get_config, Settings

@pytest.fixture(scope="session")
def config():
    """Provide test configuration."""
    return get_config()

@pytest.fixture
def settings():
    """Provide test settings (override with test env vars)."""
    return Settings(
        neo4j_uri="bolt://localhost:7687",
        neo4j_user="neo4j",
        neo4j_password="test_password",
        openai_api_key="sk-test-key"
    )
```

---

*Testing analysis: 2026-04-01*

## Current Status

**Backend (Python):**
- Testing infrastructure installed: pytest, pytest-asyncio, pytest-cov
- **No test files written** - .pytest_cache exists but empty
- Recommendation: Start with unit tests for authority scoring (most complex logic)

**Frontend (TypeScript):**
- ESLint configured for static analysis
- **No test runner or test files** - use this to catch compilation errors
- Recommendation: Add Jest or Vitest for React component testing when needed
