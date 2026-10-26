---
last_mapped_commit: b97d1fef40a1301a65818a1554fd2d67cc274d27
last_mapped_at: 2026-10-26
---
# Codebase Structure

**Analysis Date:** 2026-10-26

## Directory Layout

```
finally/                             # Project root
├── backend/                         # FastAPI uv project (Python)
│   ├── app/                         # Application code
│   │   ├── __init__.py
│   │   └── market/                  # Market data subsystem [COMPLETE]
│   │       ├── __init__.py          # Public API exports
│   │       ├── models.py            # PriceUpdate dataclass
│   │       ├── interface.py         # MarketDataSource ABC
│   │       ├── cache.py             # PriceCache (thread-safe)
│   │       ├── seed_prices.py       # Seed data, correlation params
│   │       ├── simulator.py         # GBM simulator + SimulatorDataSource
│   │       ├── massive_client.py    # Massive API client
│   │       ├── factory.py           # create_market_data_source()
│   │       └── stream.py            # create_stream_router() for SSE
│   │
│   ├── routes/                      # API endpoints [PLANNED]
│   │   ├── __init__.py
│   │   ├── portfolio.py             # /api/portfolio, /api/portfolio/trade
│   │   ├── watchlist.py             # /api/watchlist
│   │   ├── chat.py                  # /api/chat (LLM integration)
│   │   └── health.py                # /api/health
│   │
│   ├── db/                          # Database layer [PLANNED]
│   │   ├── __init__.py
│   │   ├── schema.sql               # SQLite DDL (or alembic migrations)
│   │   ├── seed.py                  # Seed data + lazy init
│   │   └── models.py                # ORM models or query builders
│   │
│   ├── llm/                         # LLM integration [PLANNED]
│   │   ├── __init__.py
│   │   ├── client.py                # LiteLLM via OpenRouter
│   │   └── structured_output.py     # Parse trade/watchlist actions
│   │
│   ├── main.py                      # FastAPI app entry point [PLANNED]
│   ├── conftest.py                  # Pytest fixtures (if at top level)
│   ├── pyproject.toml               # uv project config
│   ├── uv.lock                      # Lockfile (committed)
│   ├── CLAUDE.md                    # Backend developer guide
│   ├── README.md                    # Backend setup instructions
│   │
│   ├── tests/                       # Unit tests
│   │   ├── __init__.py
│   │   ├── conftest.py              # Event loop, shared fixtures
│   │   ├── market/                  # Market data tests [COMPLETE]
│   │   │   ├── test_models.py       # 11 tests, 100% coverage
│   │   │   ├── test_cache.py        # 13 tests, 100% coverage
│   │   │   ├── test_simulator.py    # 17 tests, 98% coverage
│   │   │   ├── test_simulator_source.py  # 10 integration tests
│   │   │   ├── test_factory.py      # 7 tests, 100% coverage
│   │   │   ├── test_massive.py      # 13 tests, 56% coverage (API mocked)
│   │   │   └── __init__.py
│   │   ├── routes/                  # Route tests [PLANNED]
│   │   ├── db/                      # Database layer tests [PLANNED]
│   │   └── llm/                     # LLM integration tests [PLANNED]
│   │
│   ├── market_data_demo.py          # Rich terminal dashboard (runnable script)
│
├── frontend/                        # Next.js TypeScript project [PLANNED]
│   ├── src/
│   │   ├── components/              # React components
│   │   ├── pages/                   # Next.js pages / app routes
│   │   ├── hooks/                   # Custom React hooks (SSE, data fetching)
│   │   └── styles/                  # Tailwind + custom CSS
│   ├── public/                      # Static assets
│   ├── package.json
│   ├── next.config.js
│   ├── tsconfig.json
│   └── [tests/]                     # React Testing Library / Vitest
│
├── test/                            # E2E tests with Playwright [PLANNED]
│   ├── e2e/                         # Test specs
│   ├── fixtures/                    # Mock data
│   ├── docker-compose.test.yml      # Isolated test environment
│   ├── playwright.config.ts
│   └── package.json
│
├── scripts/                         # Start/stop scripts [PLANNED]
│   ├── start_mac.sh                 # Docker run wrapper (macOS/Linux)
│   ├── stop_mac.sh
│   ├── start_windows.ps1            # Docker run wrapper (Windows)
│   └── stop_windows.ps1
│
├── db/                              # Runtime SQLite volume mount point
│   └── .gitkeep                     # Directory exists; finally.db is gitignored
│
├── planning/                        # Project documentation for agents
│   ├── PLAN.md                      # Full project specification
│   ├── MARKET_DATA_SUMMARY.md       # Market data implementation summary
│   └── archive/                     # Previous phase docs
│
├── .planning/codebase/              # This is where you are
│   ├── ARCHITECTURE.md
│   ├── STRUCTURE.md
│   ├── [STACK.md]
│   ├── [INTEGRATIONS.md]
│   ├── [CONVENTIONS.md]
│   ├── [TESTING.md]
│   └── [CONCERNS.md]
│
├── .claude/                         # GSD workflow tooling (ignore)
├── .github/
├── .gitignore
├── CLAUDE.md                        # Project instructions
├── Dockerfile                       # Multi-stage build [PLANNED]
├── docker-compose.yml               # Convenience wrapper [PLANNED]
├── .env.example                     # Sample env vars
├── README.md                        # Project overview
└── LICENSE
```

## Directory Purposes

**`backend/app/`** — Core application logic
- Contains the complete market data subsystem and planned API, database, and LLM layers
- Organized by concern: market data, routes, database, LLM
- All business logic lives here

**`backend/app/market/`** — Market data subsystem [COMPLETE, 500 lines]
- Price models, cache, simulator, Massive API client
- Fully tested (73 tests, 84% coverage)
- No external dependencies except numpy and the `massive` package
- Ready for production use

**`backend/routes/`** — API endpoints [PLANNED]
- REST routes for portfolio, watchlist, chat, health
- Depends on market data cache and database layer
- Validates requests with Pydantic, returns JSON

**`backend/db/`** — Database layer [PLANNED]
- SQLite schema, seed data, ORM or query builders
- Lazy initialization on first request
- User state: positions, trades, watchlist, portfolio snapshots, chat history

**`backend/llm/`** — LLM integration [PLANNED]
- LiteLLM client to OpenRouter
- Structured output parsing (trades, watchlist changes)
- Auto-execution of trade/watchlist commands

**`backend/tests/market/`** — Market data test suite [COMPLETE]
- 73 tests across 6 modules
- 84% overall coverage
- Unit tests (models, cache, simulator) and integration tests (sources)

**`frontend/`** — Next.js React app [PLANNED]
- Static export to `out/` for serving by FastAPI
- SSE client connects to `/api/stream/prices`
- REST calls to `/api/*` for trades, watchlist, chat

**`test/`** — E2E tests [PLANNED]
- Playwright + docker-compose.test.yml
- Full system testing without mocking
- Run with `LLM_MOCK=true` for determinism

**`scripts/`** — Start/stop wrappers [PLANNED]
- Bash scripts for macOS/Linux
- PowerShell scripts for Windows
- Idempotent, one-command startup

**`planning/`** — Agent documentation
- `PLAN.md` — full spec, shared contract for all phases
- `MARKET_DATA_SUMMARY.md` — what's been built
- `archive/` — previous phase docs

**`.planning/codebase/`** — This directory
- ARCHITECTURE.md, STRUCTURE.md, CONVENTIONS.md, TESTING.md, STACK.md, INTEGRATIONS.md, CONCERNS.md
- Read by `/gsd-plan-phase` and `/gsd-execute-phase` to guide implementation

## Key File Locations

**Entry Points:**
- `backend/main.py` (FastAPI app) — [TO BE CREATED]
- `backend/app/market/stream.py` — SSE endpoint router [COMPLETE]
- `frontend/pages/index.tsx` or `app/page.tsx` — browser entry point [PLANNED]

**Configuration:**
- `backend/pyproject.toml` — uv dependencies, test/lint config
- `backend/.env` — environment variables (gitignored)
- `frontend/next.config.js` — Next.js config with static export [PLANNED]

**Core Logic:**
- `backend/app/market/cache.py` — PriceCache implementation [COMPLETE]
- `backend/app/market/interface.py` — MarketDataSource ABC [COMPLETE]
- `backend/app/market/simulator.py` — GBM simulator [COMPLETE]
- `backend/routes/*.py` — API endpoints [PLANNED]
- `backend/db/schema.sql` — SQLite DDL [PLANNED]

**Testing:**
- `backend/tests/conftest.py` — pytest configuration
- `backend/tests/market/test_*.py` — market data tests [6 modules, 73 tests]
- `test/playwright.config.ts` — E2E test config [PLANNED]

**Documentation:**
- `CLAUDE.md` — project instructions (this file)
- `planning/PLAN.md` — full spec
- `backend/CLAUDE.md` — backend developer guide
- `backend/README.md` — backend setup

## Naming Conventions

**Files:**
- Module files (Python): `lowercase_with_underscores.py` — `cache.py`, `interface.py`, `simulator.py`
- Route files: `feature_name.py` — `portfolio.py`, `watchlist.py`, `chat.py`
- Test files: `test_module_name.py` — `test_cache.py`, `test_simulator.py`
- React components: `PascalCase.tsx` — `WatchlistPanel.tsx`, `PortfolioHeatmap.tsx`

**Directories:**
- Feature layers: `lowercase` — `market`, `routes`, `db`, `llm`, `tests`
- React: `lowercase` — `components`, `pages`, `hooks`, `styles`
- E2E: `e2e`, `fixtures`

**Python Classes:**
- `PascalCase` — `PriceUpdate`, `PriceCache`, `MarketDataSource`, `SimulatorDataSource`

**Python Functions:**
- `lowercase_with_underscores` — `create_market_data_source()`, `stream_prices()`, `update()`

**Python Constants:**
- `UPPERCASE_WITH_UNDERSCORES` — `TRADING_SECONDS_PER_YEAR`, `DEFAULT_PARAMS`, `SEED_PRICES`

**Database:**
- Table names: `snake_case` — `users_profile`, `portfolio_snapshots`, `chat_messages`
- Column names: `snake_case` — `user_id`, `cash_balance`, `created_at`

**API Endpoints:**
- Path segments: `lowercase` with hyphens for multi-word — `/api/portfolio`, `/api/stream/prices`, `/api/chat`
- Query params: `camelCase` or `lowercase` — `?ticker=AAPL&side=buy`

**Environment Variables:**
- `UPPERCASE_WITH_UNDERSCORES` — `OPENROUTER_API_KEY`, `MASSIVE_API_KEY`, `LLM_MOCK`

## Where to Add New Code

**New Feature (e.g., Alerts):**
- Backend implementation: Create `backend/app/alerts/` with models, logic, tests
- API route: `backend/routes/alerts.py`
- Database: Add tables to `backend/db/schema.sql`
- Frontend: Add `frontend/src/components/AlertsPanel.tsx`
- Tests: `backend/tests/alerts/test_*.py` and `test/e2e/alerts.spec.ts`

**New Component/Module:**
- Python module: `backend/app/{name}/` with `__init__.py` exporting public API
- Tests: `backend/tests/{name}/`
- React component: `frontend/src/components/{Name}.tsx`

**Utilities:**
- Shared Python helpers: `backend/app/utils.py` (keep small; split into modules if >200 lines)
- Shared React hooks: `frontend/src/hooks/{name}.ts`
- Shared constants: `backend/app/constants.py`, `frontend/src/constants.ts`

**New API Endpoint:**
1. Create route function in `backend/routes/{feature}.py`
2. Register router in `backend/main.py` — `app.include_router(portfolio_router, prefix="/api")`
3. Add Pydantic request/response models if needed
4. Add tests in `backend/tests/routes/test_{feature}.py`
5. Document in PLAN.md if user-facing

## Special Directories

**`backend/app/market/`** — Market Data Subsystem
- Purpose: Self-contained, reusable module for price streaming
- Generated: No (all source code)
- Committed: Yes
- Mature: Complete, 84% test coverage, production-ready
- Notes: Used as template for structure/patterns in other modules (routes, db, llm)

**`backend/tests/`** — Unit and Integration Tests
- Purpose: Verify backend logic without frontend or E2E machinery
- Generated: No (all source code)
- Committed: Yes
- Run: `cd backend && uv run --extra dev pytest -v`

**`test/`** — E2E Tests (Planned)
- Purpose: Full system testing via browser automation
- Generated: No (tests are source code, but test artifacts are generated)
- Committed: Yes (tests, config, docker-compose.test.yml)
- Run: `docker-compose -f test/docker-compose.test.yml up` or `npx playwright test`

**`frontend/out/`** — Static Export (Planned)
- Purpose: Built Next.js static files to be served by FastAPI
- Generated: Yes (by `npm run build`)
- Committed: No (in .gitignore)
- Location in container: Copied into FastAPI static serving directory during Docker build

**`db/`** — SQLite Runtime Directory
- Purpose: Volume mount point for persistent database file
- Generated: Yes (by backend on first request)
- Committed: No (.gitkeep only; finally.db is gitignored)
- Usage: `docker run -v finally-data:/app/db ...` mounts this to `/app/db` in container

**`.env`** — Environment Variables (Runtime)
- Purpose: Configuration not in code (API keys, secrets)
- Generated: No (user creates from .env.example)
- Committed: No (gitignored)
- Caution: Never commit actual API keys; use .env.example

---

*Structure analysis: 2026-10-26*
