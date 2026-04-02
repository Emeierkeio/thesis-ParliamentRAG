# Technology Stack

**Analysis Date:** 2026-04-01

## Languages

**Primary:**
- Python 3.13 - Backend API, data processing, ML services
- TypeScript 5 - Frontend application with React
- JavaScript (ES2017) - Build tooling

**Secondary:**
- Cypher - Neo4j graph query language
- YAML - Configuration files

## Runtime

**Environment:**
- Node.js (used by Next.js, version specified via Next.js 16.1.4)
- Python 3.13 (inferred from venv in `backend/venv/lib/python3.13`)
- Uvicorn ASGI server for FastAPI

**Package Manager:**
- npm (frontend) - See `frontend/package.json` for dependencies
- pip (backend) - See `backend/requirements.txt` for dependencies
- Lockfile: `package-lock.json` present for frontend

## Frameworks

**Core:**
- FastAPI 0.109.0+ - REST API framework (Python backend)
- Next.js 16.1.4 - React SSR/static generation framework (TypeScript frontend)
- React 19.2.3 - Frontend UI library

**Database:**
- Neo4j 5.15.0 - Graph database with vector search capabilities
- Python neo4j driver 5.15.0+ - Neo4j client

**Styling & UI Components:**
- Tailwind CSS 4 - Utility-first CSS framework
- Radix UI (multiple components) - Headless UI component library
  - `@radix-ui/react-avatar`, `@radix-ui/react-dialog`, `@radix-ui/react-tabs`, `@radix-ui/react-popover`, etc.
- class-variance-authority 0.7.1 - CSS class composition
- tailwind-merge 3.4.0 - Merge Tailwind class conflicts

**AI/ML:**
- OpenAI Python client (openai >= 1.10.0) - LLM inference and embeddings
- spacy >= 3.5.0 - NLP processing

**Testing:**
- pytest >= 7.4.0 - Python test framework
- pytest-asyncio >= 0.23.0 - Async test support
- pytest-cov >= 4.1.0 - Coverage reporting

**Build/Dev:**
- TypeScript 5 - Type checking (frontend)
- ESLint 9 - JavaScript/TypeScript linting
- eslint-config-next 16.1.4 - Next.js ESLint rules
- black >= 24.1.0 - Python code formatter
- isort >= 5.13.0 - Python import sorter
- mypy >= 1.8.0 - Python type checker
- tailwindcss 4 - CSS generation from Tailwind directives

## Key Dependencies

**Critical:**
- openai >= 1.10.0 - **Only LLM provider** for embeddings and generation (not Claude/Anthropic)
- neo4j >= 5.15.0 - Graph database driver with vector index support
- fastapi >= 0.109.0 - Backend REST API framework
- uvicorn[standard] >= 0.27.0 - ASGI server with streaming support

**Infrastructure:**
- pydantic >= 2.5.0 - Data validation and settings management
- pydantic-settings >= 2.1.0 - Environment-based configuration
- python-dotenv >= 1.0.0 - Environment variable loading
- PyYAML >= 6.0.1 - YAML config file parsing
- httpx >= 0.26.0 - Async HTTP client (for internal requests)
- python-multipart >= 0.0.6 - Multipart form data handling
- numpy >= 1.24.0 - Numerical computing for embeddings
- scipy >= 1.11.0 - Scientific computing (authority calculations)

**Frontend UI:**
- lucide-react 0.563.0 - Icon library
- clsx 2.1.1 - Conditional CSS class composition
- react-markdown 10.1.0 - Markdown rendering
- cmdk 1.1.1 - Command palette component
- next-themes 0.4.6 - Dark mode theme management
- react-force-graph-2d 1.29.0 - 2D force-directed graph visualization

## Configuration

**Environment:**
- `.env` file location: Project root or `backend/` directory
- Config file: `backend/config/default.yaml` - All weights, thresholds, settings
- Commission topics mapping: `backend/config/commissioni_topics.yaml`

**Key Environment Variables:**
- `NEO4J_URI` - Neo4j Bolt connection URI (default: `bolt://localhost:7689`)
- `NEO4J_USER` - Neo4j username (default: `neo4j`)
- `NEO4J_PASSWORD` - Neo4j password (REQUIRED, no default)
- `OPENAI_API_KEY` - OpenAI API key (REQUIRED) - single key or comma-separated list for rate limit distribution
- `NEXT_PUBLIC_API_URL` - Frontend API base URL (default: `http://localhost:8000/api`)
- `ENVIRONMENT` - Runtime environment (development, staging, production)
- `LOG_LEVEL` - Logging verbosity (DEBUG, INFO, WARNING, ERROR)
- `WORKERS` - Number of Uvicorn workers (default: 4)

**Build:**
- `frontend/tsconfig.json` - TypeScript configuration with `@/*` path alias
- `frontend/next.config.ts` - Next.js config with `output: 'standalone'` for Docker
- `.env.example` - Template for required environment variables

## Platform Requirements

**Development:**
- Python 3.13+ with pip
- Node.js (version managed by Next.js ecosystem)
- Neo4j 5.15.0+ running on specified URI
- OpenAI API account with valid API key

**Production:**
- Deployment target: Docker containerized environment
  - Backend: `backend/Dockerfile` - FastAPI app served by Uvicorn
  - Database: Docker Compose with neo4j:5.15.0 service (see `docker-compose.yml`)
- HTTPS recommended for OpenAI API calls and frontend deployment
- Appropriate firewall rules for Neo4j Bolt protocol (default: 7687)

---

*Stack analysis: 2026-04-01*
