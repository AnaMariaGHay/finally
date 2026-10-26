---
last_mapped_commit: b97d1fef40a1301a65818a1554fd2d67cc274d27
last_mapped_at: 2026-10-26
---
# External Integrations

**Analysis Date:** 2026-10-26

## APIs & External Services

**Market Data (Conditional):**
- Massive (Polygon.io) - Real-time stock market data via REST API
  - What it's used for: Optional alternative to GBM simulator; provides actual market prices for watched tickers
  - SDK/Client: `massive` Python package (v1.0.0+) via `MassiveDataSource` in `backend/app/market/massive_client.py`
  - Auth: `MASSIVE_API_KEY` environment variable (Polygon.io API key)
  - When active: If `MASSIVE_API_KEY` is set and non-empty in environment
  - When inactive: If `MASSIVE_API_KEY` is absent or empty; system falls back to built-in GBM simulator
  - API endpoint: `GET /v2/snapshot/locale/us/markets/stocks/tickers` (REST polling)
  - Rate limits:
    - Free tier: 5 req/min → polls every 15 seconds (default)
    - Paid tiers: higher limits → polls every 2-5 seconds (configurable)
  - Implementation: `MassiveDataSource` class runs async polling loop, updating `PriceCache` with latest snapshots

**LLM Integration (Planned):**
- OpenRouter - API gateway to multiple LLM providers
  - What it's used for: AI chat assistant for portfolio analysis, trade suggestions, and natural language trade execution (planned)
  - SDK/Client: LiteLLM (not yet added to dependencies) via OpenRouter API gateway
  - Model: `openrouter/openai/gpt-oss-120b` with Cerebras as inference provider
  - Auth: `OPENROUTER_API_KEY` environment variable (OpenRouter account API key)
  - Implementation: Not yet implemented; see planning/PLAN.md § 9 for intended behavior (structured JSON outputs, auto-trade execution)
  - When active: When `/api/chat` endpoint is called (planned)
  - Features: Structured outputs for parsing LLM responses; auto-execution of trades and watchlist changes

## Data Storage

**Databases:**
- SQLite (planned)
  - Provider: Embedded, self-hosted
  - Connection: File-based at `db/finally.db` (mounted via Docker volume in production)
  - Client: Python standard library `sqlite3` (no ORM dependency as of now)
  - Lazy initialization: Database is created on first request if not present; tables and seed data auto-populated
  - Persistence: Docker named volume `finally-data` → `/app/db/` in container
  - Schema (planned):
    - `users_profile` - User state (cash balance, user_id)
    - `watchlist` - Watched tickers per user
    - `positions` - Current holdings (ticker, quantity, avg cost per user)
    - `trades` - Trade history (append-only log per user)
    - `portfolio_snapshots` - Portfolio value over time (P&L tracking per user)
    - `chat_messages` - Conversation history with LLM (message, role, auto-executed actions per user)
  - Multi-user ready: All tables include `user_id` column; currently hardcoded to `"default"` for single-user mode

**File Storage:**
- Local filesystem only (no external cloud storage)
  - Frontend: Static files (Next.js export) served from `backend/` directory by FastAPI
  - Database: SQLite file at `db/finally.db`
  - Logs: stdout/stderr (managed by container runtime)

**Caching:**
- In-memory price cache (`PriceCache` in `backend/app/market/cache.py`)
  - Thread-safe, no external cache service
  - Holds latest price, previous price, timestamp per ticker
  - Used by SSE stream, portfolio valuation, trade execution

## Authentication & Identity

**Auth Provider:**
- Custom / None (currently)
  - Implementation: No authentication layer; single hardcoded user `"default"` in database
  - Future multi-user: Database schema ready; user_id defaults to `"default"` but can be parameterized

**API Key Management:**
- Environment variables for third-party API keys:
  - `MASSIVE_API_KEY` - Polygon.io key, optional
  - `OPENROUTER_API_KEY` - OpenRouter key, required for LLM chat (not yet used)
  - `.env` file at project root (gitignored); `.env.example` should be committed with placeholder values
  - Docker: Keys passed via `--env-file .env` at runtime

## Monitoring & Observability

**Error Tracking:**
- None detected (not implemented)
- Planned: No external error tracking tool mentioned in project plan

**Logs:**
- Python `logging` module (standard library)
  - Output: stdout/stderr (standard for Docker containers)
  - Format: Not yet standardized; current code uses `logging.getLogger(__name__)`
  - Levels: INFO (startup/shutdown), DEBUG (polling details), WARNING (parse errors, API failures), ERROR (critical failures)
  - Used in: `backend/app/market/massive_client.py`, `backend/app/market/simulator.py`, `backend/app/market/cache.py`, `backend/app/market/stream.py`

**Performance Monitoring:**
- None yet; SSE stream logs client connections/disconnections in `backend/app/market/stream.py`

## CI/CD & Deployment

**Hosting:**
- Docker container (single container, single port 8000)
  - Dockerfile: Not yet implemented (multi-stage: Node 20 build → Python 3.12 runtime)
  - Image: Single image serving FastAPI + static frontend files
  - Registry: Not specified; assume Docker Hub or container registry of choice

**CI Pipeline:**
- None detected yet
- Planned: GitHub Actions workflows for testing, linting, Docker build/push (mentioned in project structure but `.github/` exists with no workflows yet)

**Deployment Method:**
- Docker Compose (optional convenience wrapper)
  - File: Not yet implemented; planned at project root
- Direct Docker run with volume mount for persistence
- Candidate platforms: AWS App Runner, Render, Heroku, any Docker-hosting service
- Database persistence: Named Docker volume `finally-data` mapped to `db/` in container

## Environment Configuration

**Required Environment Variables:**
- `OPENROUTER_API_KEY` - OpenRouter API key for LLM integration (chat functionality requires this)

**Optional Environment Variables:**
- `MASSIVE_API_KEY` - Polygon.io API key for real market data; if not set, GBM simulator is used
- `LLM_MOCK` - Set to `"true"` for deterministic mock LLM responses (for E2E testing)

**Configuration Files:**
- `.env` - Environment variables (gitignored)
- `.env.example` - Template with placeholder values (should be committed)
- `backend/pyproject.toml` - Python dependencies and configuration

**Secrets Location:**
- `.env` file at project root (local development)
- Docker: via `--env-file .env` at runtime or via container orchestration secrets (AWS Secrets Manager, Kubernetes Secrets, etc.)
- CI/CD: GitHub Actions secrets (when implemented)

## Webhooks & Callbacks

**Incoming:**
- SSE stream only (no HTTP POST webhooks incoming); SSE is one-way server → client push
- Endpoint: `GET /api/stream/prices` returns text/event-stream

**Outgoing:**
- None yet; planned LLM integration will not use webhooks (synchronous chat API)
- No Webhook.cool, Zapier, or IFTTT integrations

## Trade Execution & Market Orders

**Execution Method:**
- Simulated orders (instant market fill at current SSE price)
- No real trading (paper trading on $10k virtual cash)
- Via `/api/portfolio/trade` endpoint (planned) and `/api/chat` endpoint (planned)

**Price Feed for Execution:**
- Current price from `PriceCache` (fed by Massive API or GBM simulator)
- No order book, no limit orders, no partial fills

## Data Pipeline

**Market Data Flow:**

```
Massive API or GBM Simulator
    ↓
PriceCache (thread-safe in-memory store)
    ├→ SSE stream `/api/stream/prices` (to frontend)
    ├→ Portfolio valuation (in trade execution)
    └→ Watchlist display (with latest prices)
```

**Trade Execution Flow:**

```
/api/portfolio/trade endpoint (planned)
    ↓
Validate: sufficient cash/shares, market price from PriceCache
    ↓
Execute trade: update positions, update cash balance (planned)
    ↓
Record: append to trades table, record portfolio_snapshot (planned)
    ↓
Response: success/error to client
```

**Chat + Trade Flow:**

```
/api/chat endpoint (planned)
    ↓
OpenRouter LLM API (via LiteLLM)
    ↓
Parse structured JSON response
    ↓
Auto-execute any trades in response
    ↓
Record message + actions in chat_messages table
    ↓
Return complete response to frontend
```

---

*Integration audit: 2026-10-26*
