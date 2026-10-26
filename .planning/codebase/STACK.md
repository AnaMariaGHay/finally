---
last_mapped_commit: b97d1fef40a1301a65818a1554fd2d67cc274d27
last_mapped_at: 2026-10-26
---
# Technology Stack

**Analysis Date:** 2026-10-26

## Languages

**Primary:**
- Python 3.12+ - Backend application and market data engine (`backend/`)

**Secondary:**
- TypeScript/JavaScript - Frontend (Next.js) — planned, not yet implemented

## Runtime

**Environment:**
- Python 3.12+ (required in `backend/pyproject.toml`)

**Package Manager:**
- uv (modern, fast Python package manager)
- Lockfile: `backend/uv.lock` — committed to git, ensures reproducible builds

## Frameworks

**Core:**
- FastAPI (≥0.115.0) - REST API framework and application server, partially implemented
- Uvicorn (≥0.32.0) - ASGI server for FastAPI (with `standard` extras for performance features like uvloop, httptools)

**Testing:**
- pytest (≥8.3.0) - Unit and integration test runner
- pytest-asyncio (≥0.24.0) - Async test support for FastAPI endpoints
- pytest-cov (≥5.0.0) - Code coverage reporting

**Build/Dev:**
- ruff (≥0.7.0) - Fast Python linter and formatter; configured in `backend/pyproject.toml`
- hatchling - Build backend for Python packaging

## Key Dependencies

**Critical:**
- numpy (≥2.0.0) - Used in the GBM (Geometric Brownian Motion) simulator for correlated price generation in `backend/app/market/simulator.py`
- massive (≥1.0.0) - Polygon.io REST client SDK for real market data streaming via `MassiveDataSource` in `backend/app/market/massive_client.py`
- rich (≥13.0.0) - Rich terminal UI library; currently used in `backend/market_data_demo.py` for the Rich terminal dashboard

**Core Infrastructure:**
- fastapi - Web framework and routing
- uvicorn[standard] - ASGI application server with optimized dependencies
- pydantic (via FastAPI) - Data validation and serialization
- starlette (via FastAPI) - HTTP utilities and streaming support

**Development:**
- ruff - Linting and code formatting

## Configuration

**Environment Variables:**
- `MASSIVE_API_KEY` (optional) - Polygon.io API key; if set, enables real market data via Massive API; if absent, uses built-in GBM simulator
- `OPENROUTER_API_KEY` (required) - OpenRouter API key for LLM chat integration (planned, not yet implemented)
- `LLM_MOCK` (optional) - Set to `"true"` for deterministic mock LLM responses in testing (planned, not yet implemented)

**Build Configuration:**
- `backend/pyproject.toml` - Python project manifest with all dependencies, test configuration, and ruff linting rules
- `backend/uv.lock` - Reproducible lock file for all transitive dependencies

**Linting:**
- `backend/pyproject.toml` includes ruff configuration:
  - Line length: 100
  - Target Python version: 3.12
  - Selected rules: E (errors), F (pyflakes), I (imports), N (naming), W (warnings)
  - Ignored: E501 (line too long — handled by formatter)

**Test Configuration:**
- `backend/pyproject.toml` pytest configuration:
  - Test discovery paths: `tests/`
  - Test file pattern: `test_*.py`
  - Test class pattern: `Test*`
  - Test function pattern: `test_*`
  - Async mode: auto (all tests run with asyncio support)
  - Coverage source: `app/` only (not tests)

## Platform Requirements

**Development:**
- Python 3.12+ (or later) installed on local machine
- uv package manager (`pip install uv`)
- Optional: Massive API key for real market data (Polygon.io account)
- Optional: OpenRouter API key for LLM chat (OpenRouter account)

**Production:**
- Single Docker container (multi-stage: Node build stage for Next.js frontend, Python runtime for FastAPI backend)
- Deployment target: Any platform supporting Docker containers (AWS App Runner, Render, Heroku, etc.)
- Single port: 8000 (FastAPI serves both API routes and static frontend files)
- SQLite database file persisted via Docker named volume (`finally-data`) mounted to `/app/db/`

## Current Implementation Status

**✓ Complete:**
- Backend market data subsystem (`backend/app/market/`) — 8 modules, ~500 lines
  - GBM simulator with correlated price generation
  - Massive API client for real market data
  - SSE streaming endpoint at `/api/stream/prices`
  - Price cache (thread-safe, in-memory)
- Test suite (73 tests, 84% coverage)
- Market data demo CLI (`backend/market_data_demo.py`)

**✗ Not Yet Implemented:**
- Frontend (Next.js static export) — planned in `frontend/` directory
- Main FastAPI application entrypoint
- SQLite database schema and lazy initialization
- REST API endpoints for portfolio, watchlist, trades, chat
- LLM integration (OpenRouter via LiteLLM)
- Docker multi-stage build and Docker Compose
- Start/stop shell scripts (`scripts/start_mac.sh`, `start_windows.ps1`, etc.)
- E2E tests (Playwright in `test/` directory)
- API endpoints:
  - `/api/portfolio` - GET/portfolio state, POST/trade execution
  - `/api/watchlist` - GET/list, POST/add, DELETE/remove
  - `/api/chat` - POST/send message with LLM response and auto-trade execution
  - `/api/portfolio/history` - GET/P&L snapshots for charting
  - `/api/health` - Health check for deployment

---

*Stack analysis: 2026-10-26*
