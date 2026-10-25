# FinAlly — AI Trading Workstation

An AI-powered trading terminal: live streaming prices, a simulated $10k portfolio, and a chat assistant that can analyze your positions and place trades for you. Built entirely by coding agents as the capstone for an agentic AI coding course.

> **Status:** the market data subsystem (`backend/app/market/`) is complete. The rest of the platform is in progress. See [`planning/PLAN.md`](planning/PLAN.md).

## Features

- Live prices over SSE, with green/red flash animations and sparklines
- Market orders with instant fills, fractional shares, and no fees
- Portfolio heatmap, P&L chart, and positions table
- AI chat that executes trades and manages your watchlist
- Dark, Bloomberg-style layout

## Stack

One Docker container on port 8000:

- **Frontend:** Next.js + TypeScript + Tailwind, built as a static export
- **Backend:** FastAPI (Python, `uv`) serving the REST, SSE, and static files
- **Database:** SQLite, created and seeded at startup
- **AI:** LiteLLM → OpenRouter (`gpt-oss-120b` on Cerebras) with structured outputs
- **Market data:** a built-in GBM simulator by default, or the Massive (Polygon.io) API if you provide a key

## Quick Start

```bash
cp .env.example .env          # then set OPENROUTER_API_KEY
./scripts/start_mac.sh        # Windows: .\scripts\start_windows.ps1
# open http://localhost:8000
```

To stop it, run `./scripts/stop_mac.sh` (or `stop_windows.ps1`). Your data stays in the `finally-data` Docker volume.

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `OPENROUTER_API_KEY` | Yes | Powers the AI chat |
| `MASSIVE_API_KEY` | No | Uses real market data; leave it empty to use the simulator |
| `LLM_MOCK` | No | Set to `true` for deterministic mock AI responses (testing) |
| `DB_PATH` | No | SQLite file path (default: `db/finally.db`) |

## Development

```bash
cd backend
uv sync
uv run pytest                    # unit tests
uv run market_data_demo.py       # terminal demo of the price simulator
```

## Layout

```
frontend/   Next.js app (static export)
backend/    FastAPI uv project (app/, tests/)
planning/   Specs and agent docs. Start with PLAN.md
scripts/    Docker start/stop scripts
test/       Playwright E2E tests
db/         SQLite file for local runs
```

## License

See [LICENSE](LICENSE).
