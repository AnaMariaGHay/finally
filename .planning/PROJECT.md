# FinAlly — AI Trading Workstation

## What This Is

FinAlly (Finance Ally) is a visually stunning, Bloomberg-terminal-style trading workstation that streams live (simulated or real) market data, lets a single user trade a $10,000 simulated portfolio, and embeds an LLM chat copilot that analyzes positions and executes trades and watchlist changes through natural language. It is the capstone project for an agentic AI coding course — built entirely by coding agents to demonstrate that orchestrated agents can ship a production-quality full-stack app. The full product spec lives in `planning/PLAN.md` and is the shared contract.

## Core Value

A user runs one Docker command, opens `localhost:8000`, and immediately sees live-streaming prices, can buy/sell instantly, and can ask the AI copilot to analyze and trade for them — all working end-to-end in a single container.

## Requirements

### Validated

<!-- Inferred from existing code (backend/app/market/, 73 tests, 84% coverage). -->

- ✓ Abstract `MarketDataSource` interface with two interchangeable implementations — existing
- ✓ GBM price simulator: ~500ms updates, correlated moves, random 2–5% events, realistic seed prices — existing
- ✓ Massive (Polygon.io) REST poller selected when `MASSIVE_API_KEY` is set — existing
- ✓ Thread-safe in-memory `PriceCache` (latest price, previous price, timestamp, version counter) — existing
- ✓ SSE endpoint `GET /api/stream/prices` pushing ticker/price/previous/timestamp/direction (router factory, not yet mounted in an app) — existing

### Active

**Backend platform**
- [ ] FastAPI app entrypoint wiring market data source, SSE router, API routers, background tasks, and static file serving on port 8000
- [ ] SQLite at `db/finally.db` with lazy init: schema for users_profile, watchlist, positions, trades, portfolio_snapshots, chat_messages (all with `user_id="default"`), seeded with $10,000 cash and 10 default tickers (AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX)
- [ ] `GET /api/health`
- [ ] Watchlist API: GET (tickers with latest prices), POST `{ticker}`, DELETE `/{ticker}`; market data source tracks watchlist changes
- [ ] Portfolio API: GET (positions, cash, total value, unrealized P&L), POST `/trade` `{ticker, quantity, side}` market orders with instant fill, fractional shares, validation (insufficient cash / insufficient shares)
- [ ] `GET /api/portfolio/history` from portfolio_snapshots; snapshots recorded every 30s by background task and immediately after each trade
- [ ] Daily change % measured against session-open price (backend start / seed) for the simulator, or previous close when using Massive

**AI chat**
- [ ] `POST /api/chat`: loads portfolio + watchlist context and recent history, calls `openrouter/openai/gpt-oss-120b` via LiteLLM/OpenRouter with Cerebras provider (cerebras-inference skill), structured output `{message, trades[], watchlist_changes[]}`
- [ ] Auto-execution of LLM trades/watchlist changes through the same validation as manual trades; failures reported back in the response
- [ ] Chat messages + executed actions persisted in chat_messages
- [ ] `LLM_MOCK=true` deterministic mock responses (no API key needed)

**Frontend (Next.js + TypeScript static export, Tailwind dark theme)**
- [ ] Header: live total portfolio value, cash balance, connection status dot (green/yellow/red)
- [ ] Watchlist panel: ticker, price with green/red flash (~500ms fade), daily change %, sparkline accumulated from SSE since page load; add/remove tickers
- [ ] Main chart for selected ticker (click in watchlist selects it)
- [ ] Portfolio heatmap (treemap) sized by weight, colored by P&L
- [ ] P&L line chart of total portfolio value from snapshots
- [ ] Positions table: ticker, quantity, avg cost, current price, unrealized P&L, % change
- [ ] Trade bar: ticker, quantity, buy, sell — no confirmation dialog
- [ ] Docked/collapsible AI chat panel with history, loading indicator, inline trade/watchlist confirmations
- [ ] Visual design per spec: `#0d1117`-ish backgrounds, accent `#ecad0a`, blue `#209dd7`, purple `#753991` submit buttons; desktop-first, functional on tablet

**Packaging & testing**
- [ ] Multi-stage Dockerfile (Node 20 → Python 3.12 + uv), static export copied into backend, named volume `finally-data:/app/db`
- [ ] Idempotent start/stop scripts: `scripts/start_mac.sh`, `stop_mac.sh`, `start_windows.ps1`, `stop_windows.ps1`; optional `docker-compose.yml`
- [ ] Backend pytest coverage for portfolio math, trade validation, LLM parsing/malformed responses, API status codes and shapes
- [ ] Frontend unit tests (rendering, price flash, watchlist CRUD, portfolio calcs, chat rendering/loading)
- [ ] Playwright E2E in `test/` with `docker-compose.test.yml` (`LLM_MOCK=true`): fresh start, watchlist add/remove, buy, sell, heatmap + P&L chart, mocked AI chat trade, SSE reconnect — all green

### Out of Scope

- Authentication / signup / multi-user — single-user by design; `user_id="default"` keeps the door open
- Limit orders, order book, partial fills, fees — market orders only keeps portfolio math simple
- WebSockets — SSE one-way push is sufficient
- Token-by-token LLM streaming — Cerebras is fast enough; a loading indicator suffices
- Trade confirmation dialogs — deliberate agentic-demo choice; fake money
- Postgres / separate DB server — SQLite is zero-config for a single user
- Cloud deployment (Terraform/App Runner in `deploy/`) — stretch goal, not core build
- Historical price backfill for charts — sparklines/charts accumulate from SSE since page load

## Context

- Brownfield: `backend/` is a uv project (Python 3.12, FastAPI, numpy, massive, pytest/pytest-asyncio, ruff). Market data lives in `backend/app/market/` (8 modules). See `.planning/codebase/` and `planning/MARKET_DATA_SUMMARY.md`.
- No `frontend/`, app entrypoint, DB layer, Docker, scripts, or E2E tests exist yet.
- Course context: agents coordinate through files in `planning/`; PLAN.md is the authoritative spec.
- `.env` in project root holds `OPENROUTER_API_KEY`; optional `MASSIVE_API_KEY`, `LLM_MOCK`.
- Development host is Windows; scripts must cover both macOS/Linux and PowerShell.

## Constraints

- **Tech stack**: FastAPI + uv backend, Next.js TypeScript static export (`output: 'export'`), Tailwind, SQLite, SSE — fixed by PLAN.md
- **Deployment**: Single container, single port 8000, same-origin API (no CORS) — student-friendly one-command launch
- **LLM**: LiteLLM → OpenRouter `openrouter/openai/gpt-oss-120b` with Cerebras provider, structured outputs, via the cerebras-inference skill
- **Charts**: Canvas-based library preferred (Lightweight Charts or Recharts) for performance
- **Async safety**: DB access must not block the event loop (SSE and market data share it)
- **Testability**: E2E must run without an API key (`LLM_MOCK=true`)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Build the full PLAN.md spec as v1 | User request: "build the entire project — everything as described" | — Pending |
| v1 "done" = Docker image serves the full app on :8000 AND Playwright E2E suite (LLM_MOCK=true) passes | User-chosen acceptance bar | — Pending |
| Daily change % baseline = session-open (backend start/seed) price for simulator; prev close for Massive | Simulator has no real previous close | — Pending |
| Keep existing market data subsystem as-is; build on its interface | Complete and tested (73 tests, 84%) | — Pending |
| SSE over WebSockets, static export, SQLite, single container | Per PLAN.md rationale table | ✓ Good |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-10-27 after initialization*
