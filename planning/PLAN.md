# FinAlly — AI Trading Workstation

## Project Specification

## 1. Vision

FinAlly (Finance Ally) is a visually stunning AI-powered trading workstation that streams live market data, lets users trade a simulated portfolio, and integrates an LLM chat assistant that can analyze positions and execute trades on the user's behalf. It looks and feels like a modern Bloomberg terminal with an AI copilot.

This is the capstone project for an agentic AI coding course. It is built entirely by Coding Agents demonstrating how orchestrated AI agents can produce a production-quality full-stack application. Agents interact through files in `planning/`.

## 2. User Experience

### First Launch

The user runs a single Docker command (or a provided start script). A browser opens to `http://localhost:8000`. No login, no signup. They immediately see:

- A watchlist of 10 default tickers with live-updating prices in a grid
- $10,000 in virtual cash
- A dark, data-rich trading terminal aesthetic
- An AI chat panel ready to assist

### What the User Can Do

- **Watch prices stream** — prices flash green (uptick) or red (downtick) with subtle CSS animations that fade
- **View sparkline mini-charts** — price action beside each ticker in the watchlist, accumulated on the frontend from the SSE stream since page load (sparklines fill in progressively)
- **Click a ticker** to see a larger detailed chart in the main chart area
- **Buy and sell shares** — market orders only, instant fill at current price, no fees, no confirmation dialog
- **Monitor their portfolio** — a heatmap (treemap) showing positions sized by weight and colored by P&L, plus a P&L chart tracking total portfolio value over time
- **View a positions table** — ticker, quantity, average cost, current price, unrealized P&L, % change
- **Chat with the AI assistant** — ask about their portfolio, get analysis, and have the AI execute trades and manage the watchlist through natural language. Chat history persists across page reloads.
- **Manage the watchlist** — add/remove tickers manually or via the AI chat

### Visual Design

- **Dark theme**: backgrounds around `#0d1117` or `#1a1a2e`, muted gray borders, no pure black
- **Price flash animations**: brief green/red background highlight on price change, fading over ~500ms via CSS transitions
- **Connection status indicator**: a small colored dot (green = connected, yellow = reconnecting, red = disconnected) visible in the header
- **Professional, data-dense layout**: inspired by Bloomberg/trading terminals — every pixel earns its place
- **Responsive but desktop-first**: optimized for wide screens, functional on tablet

### Color Scheme
- Accent Yellow: `#ecad0a`
- Blue Primary: `#209dd7`
- Purple Secondary: `#753991` (submit buttons)

## 3. Architecture Overview

### Single Container, Single Port

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving        │
│                      (Next.js export)           │
│                                                 │
│  SQLite database (named Docker volume)          │
│  Background tasks: market data sim/poller,      │
│                    portfolio snapshots (30s)    │
└─────────────────────────────────────────────────┘
```

- **Frontend**: Next.js with TypeScript, built as a static export (`output: 'export'`), served by FastAPI as static files
- **Backend**: FastAPI (Python), managed as a `uv` project
- **Database**: SQLite, single file `finally.db` in the `db/` directory, persisted via a named Docker volume
- **Real-time data**: Server-Sent Events (SSE) — simpler than WebSockets, one-way server→client push, works everywhere
- **AI integration**: LiteLLM → OpenRouter (Cerebras for fast inference), with structured outputs for trade execution. One LLM call per chat message, complete JSON response, no streaming.
- **Market data**: Environment-variable driven — simulator by default, real data via Massive API if key provided

### Why These Choices

| Decision | Rationale |
|---|---|
| SSE over WebSockets | One-way push is all we need; simpler, no bidirectional complexity, universal browser support |
| Static Next.js export | Single origin, no CORS issues, one port, one container, simple deployment |
| SQLite over Postgres | No auth = no multi-user = no need for a database server; self-contained, zero config |
| Single Docker container | Students run one command; no docker-compose for production, no service orchestration |
| uv for Python | Fast, modern Python project management; reproducible lockfile; what students should learn |
| Market orders only | Eliminates order book, limit order logic, partial fills — dramatically simpler portfolio math |
| No LLM streaming | Structured output must be complete before it can be parsed; Cerebras returns in 1–3s, so a loading indicator is enough |

---

## 4. Directory Structure

```
finally/
├── frontend/                 # Next.js TypeScript project (static export)
├── backend/                  # FastAPI uv project (Python)
│   ├── app/                  # Python package (hatch builds `app`)
│   │   ├── market/           # Market data subsystem (complete — see MARKET_DATA_SUMMARY.md)
│   │   ├── db/               # Schema SQL, seed data, connection helpers
│   │   └── ...               # API routers, portfolio, LLM — structure up to Backend agent
│   └── tests/                # pytest
├── planning/                 # Project-wide documentation for agents
│   ├── PLAN.md               # This document
│   └── ...                   # Additional agent reference docs
├── scripts/
│   ├── start_mac.sh          # Launch Docker container (macOS/Linux)
│   ├── stop_mac.sh           # Stop Docker container (macOS/Linux)
│   ├── start_windows.ps1     # Launch Docker container (Windows PowerShell)
│   └── stop_windows.ps1      # Stop Docker container (Windows PowerShell)
├── test/                     # Playwright E2E tests + docker-compose.test.yml
├── db/                       # SQLite location for local (non-Docker) runs
│   └── .gitkeep              # Directory exists in repo; finally.db is gitignored
├── Dockerfile                # Multi-stage build (Node → Python)
├── .env                      # Environment variables (gitignored)
├── .env.example              # Committed template of all variables
└── .gitignore
```

There is no root `docker-compose.yml`: the start/stop scripts are the single way to launch. Compose is used only for E2E tests in `test/`.

### Key Boundaries

- **`frontend/`** is a self-contained Next.js project. It knows nothing about Python. It talks to the backend via `/api/*` endpoints and `/api/stream/*` SSE endpoints. Internal structure is up to the Frontend Engineer agent.
- **`backend/`** is a self-contained uv project with its own `pyproject.toml`. It owns all server logic including database initialization, schema, seed data, API routes, SSE streaming, market data, and LLM integration. Internal structure is up to the Backend/Market Data agents.
- **`backend/app/db/`** contains schema SQL definitions and seed logic. The backend initializes the database at startup (see §7).
- **`db/`** at the top level holds `finally.db` for local runs. In Docker, `/app/db` is backed by a named volume.
- **`planning/`** contains project-wide documentation, including this plan. All agents reference files here as the shared contract.
- **`test/`** contains Playwright E2E tests and supporting infrastructure (e.g., `docker-compose.test.yml`). Unit tests live within `frontend/` and `backend/` respectively, following each framework's conventions.
- **`scripts/`** contains start/stop scripts that wrap Docker commands.

---

## 5. Environment Variables

```bash
# Required: OpenRouter API key for LLM chat functionality
OPENROUTER_API_KEY=your-openrouter-api-key-here

# Optional: Massive (Polygon.io) API key for real market data
# If not set, the built-in market simulator is used (recommended for most users)
MASSIVE_API_KEY=

# Optional: Set to "true" for deterministic mock LLM responses (testing)
LLM_MOCK=false

# Optional: SQLite file path. Default: <repo root>/db/finally.db (Dockerfile sets /app/db/finally.db)
DB_PATH=
```

All four variables appear in the committed `.env.example`.

### Behavior

- If `MASSIVE_API_KEY` is set and non-empty → backend uses Massive REST API for market data
- If `MASSIVE_API_KEY` is absent or empty → backend uses the built-in market simulator
- If `LLM_MOCK=true` → backend returns deterministic mock LLM responses (see §9)
- **Loading:** in Docker, variables come from `docker run --env-file .env`. For local runs, the backend calls `python-dotenv`'s `load_dotenv()` on the repo-root `.env` at startup. This is a no-op when the file is absent, and it never overrides variables that are already set.

---

## 6. Market Data

> The market data subsystem is **complete** (`backend/app/market/`). See `planning/MARKET_DATA_SUMMARY.md` for usage. The only change still required is the `open_price` addition described below.

### Two Implementations, One Interface

Both the simulator and the Massive client implement the same abstract interface. The backend selects which to use based on the environment variable. All downstream code (SSE streaming, price cache, frontend) is agnostic to the source.

### Simulator (Default)

- Generates prices using geometric Brownian motion (GBM) with configurable drift and volatility per ticker
- Updates at ~500ms intervals
- Correlated moves across tickers (e.g., tech stocks move together)
- Occasional random "events" — sudden 2-5% moves on a ticker for drama
- Starts from realistic seed prices (e.g., AAPL ~$190, GOOGL ~$175, etc.); unknown tickers get a random seed price in $50–300
- Seeds the cache immediately when a ticker is added, so it's tradeable at once
- Runs as an in-process background task — no external dependencies

### Massive API (Optional)

- REST API polling (not WebSocket) — simpler, works on all tiers
- Polls for all **tracked tickers** (see below) on a configurable interval
- Free tier (5 calls/min): poll every 15 seconds
- Paid tiers: poll every 2-15 seconds depending on tier
- Parses REST response into the same format as the simulator
- A newly added ticker has no price until the next poll. Invalid tickers never receive a price and show as `null`.

### Tracked Tickers

The set of tickers the market source tracks is **watchlist ∪ open positions**:
- At startup, the backend loads both from the DB and calls `source.start(tickers)`.
- When a ticker is added to the watchlist, or bought while not tracked, call `source.add_ticker()`.
- When a ticker is removed from the watchlist, call `source.remove_ticker()` **only if** no position is held. Likewise when a position is fully sold, remove the ticker only if it isn't on the watchlist.

### Shared Price Cache

- A single background task (simulator or Massive poller) writes to an in-memory price cache
- The cache holds the latest price, previous price, timestamp, and **open price** for each ticker
- **`open_price`** (to be added) is the first price the cache records for a ticker since server start. It's the baseline for "daily change %" for both data sources, since the simulator has no real market open. It's added to `PriceUpdate` and its `to_dict()` along with `day_change_percent = (price − open_price) / open_price × 100`. It is cleared when the ticker is removed.
- SSE streams read from this cache and push updates to connected clients

### SSE Streaming

- Endpoint: `GET /api/stream/prices`
- Long-lived SSE connection; client uses native `EventSource` API. The server sends `retry: 1000` first.
- Every ~500ms the server checks the cache version and, **only if it changed**, sends **one event containing a map of all tracked tickers**:

```
data: {"AAPL": {"ticker": "AAPL", "price": 190.52, "previous_price": 190.41, "open_price": 190.00,
                "timestamp": 1761134400.123, "change": 0.11, "change_percent": 0.0578,
                "day_change_percent": 0.2737, "direction": "up"},
       "GOOGL": {...}, ...}
```

- `change` / `change_percent` / `direction` are tick-to-tick (they drive the flash animation); `day_change_percent` drives the watchlist "Chg %" column. `timestamp` is Unix seconds.
- Client handles reconnection automatically (EventSource has built-in retry)

---

## 7. Database

### SQLite Initialization at Startup

During FastAPI's lifespan startup, the backend opens `DB_PATH` (creating the parent directory if needed), runs `CREATE TABLE IF NOT EXISTS` for every table, and seeds default data if `users_profile` is empty. Startup then reads the watchlist and positions to start the market source. This means:

- No separate migration step
- No manual database setup
- Fresh Docker volumes start with a clean, seeded database automatically

### Schema

Every table except `users_profile` has a `user_id` column defaulting to `"default"` (`users_profile` uses `id` as the user key). The value is hardcoded as a single constant (`DEFAULT_USER_ID = "default"`). Don't build any multi-user plumbing, and don't thread a user parameter through functions. The column exists only so a future multi-user version needs no schema migration.

**users_profile** — User state (cash balance)
- `id` TEXT PRIMARY KEY (default: `"default"`)
- `cash_balance` REAL (default: `10000.0`)
- `created_at` TEXT (ISO timestamp)

**watchlist** — Tickers the user is watching
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `added_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**positions** — Current holdings (one row per ticker per user; rows with zero quantity are deleted, never stored)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `quantity` REAL (fractional shares supported)
- `avg_cost` REAL
- `updated_at` TEXT (ISO timestamp)
- UNIQUE constraint on `(user_id, ticker)`

**trades** — Trade history (append-only log)
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `ticker` TEXT
- `side` TEXT (`"buy"` or `"sell"`)
- `quantity` REAL (fractional shares supported)
- `price` REAL
- `executed_at` TEXT (ISO timestamp)

**portfolio_snapshots** — Portfolio value over time (for P&L chart). Recorded once at startup, every 30 seconds by a background task, and immediately after each trade execution.
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `total_value` REAL
- `recorded_at` TEXT (ISO timestamp)

**chat_messages** — Conversation history with LLM
- `id` TEXT PRIMARY KEY (UUID)
- `user_id` TEXT (default: `"default"`)
- `role` TEXT (`"user"` or `"assistant"`)
- `content` TEXT
- `actions` TEXT (JSON — the `actions` object returned by `POST /api/chat`, see §8; null for user messages)
- `created_at` TEXT (ISO timestamp)

All timestamps are ISO 8601 UTC strings (e.g. `2026-10-22T14:30:00Z`).

### Default Seed Data

- One user profile: `id="default"`, `cash_balance=10000.0`
- Ten watchlist entries: AAPL, GOOGL, MSFT, AMZN, TSLA, NVDA, META, JPM, V, NFLX

---

## 8. API Endpoints

### Conventions

- All request and response bodies are JSON.
- **Errors:** HTTP 400 (validation / business rule), 404 (not found), or 422 (malformed body, FastAPI default), always with body `{"detail": "Human-readable message"}`.
- **Tickers** are normalized to uppercase and trimmed, and must match `^[A-Z.]{1,10}$` (otherwise 400). In simulator mode any matching ticker is accepted.
- **Money** values (cash, prices, values, P&L) are rounded to 2 decimals in responses; percentages to 2 decimals; quantities to 4 decimals. The DB stores unrounded values.
- **No price yet:** if the cache has no price for a ticker, price fields are `null`. For portfolio valuation, a position without a live price is valued at its `avg_cost`.

### Market Data
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/stream/prices` | SSE stream of live price updates (format in §6) |

### Portfolio
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/portfolio` | Current positions, cash balance, total value, unrealized P&L |
| POST | `/api/portfolio/trade` | Execute a trade |
| GET | `/api/portfolio/history` | Portfolio value snapshots (last 500, oldest first) |

`GET /api/portfolio` → 200
```json
{
  "cash_balance": 8095.50,
  "positions_value": 1905.20,
  "total_value": 10000.70,
  "unrealized_pnl": 0.70,
  "positions": [
    {"ticker": "AAPL", "quantity": 10, "avg_cost": 190.45, "current_price": 190.52,
     "market_value": 1905.20, "unrealized_pnl": 0.70, "pnl_percent": 0.04}
  ]
}
```

`POST /api/portfolio/trade` with body `{"ticker": "AAPL", "quantity": 10, "side": "buy"}` → 200
```json
{
  "trade": {"id": "uuid", "ticker": "AAPL", "side": "buy", "quantity": 10, "price": 190.45,
            "executed_at": "2026-10-22T14:30:00Z"},
  "portfolio": { "...same shape as GET /api/portfolio..." }
}
```
Errors (400): `"Insufficient cash"`, `"Insufficient shares"`, `"Quantity must be at least 0.0001"`, `"Price unavailable for XYZ, try again shortly"`.

`GET /api/portfolio/history` → 200
```json
[{"recorded_at": "2026-10-22T14:30:00Z", "total_value": 10000.00}, ...]
```

### Watchlist
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/watchlist` | Current watchlist tickers with latest prices |
| POST | `/api/watchlist` | Add a ticker: `{"ticker": "PYPL"}`. Idempotent: returns 200 with the item even if it's already present |
| DELETE | `/api/watchlist/{ticker}` | Remove a ticker → 204; 404 if not on watchlist |

`GET /api/watchlist` → 200 (ordered by `added_at`)
```json
[{"ticker": "AAPL", "price": 190.52, "previous_price": 190.41, "open_price": 190.00,
  "day_change_percent": 0.27, "direction": "up"},
 {"ticker": "PYPL", "price": null, "previous_price": null, "open_price": null,
  "day_change_percent": null, "direction": null}]
```
`POST /api/watchlist` returns a single item of the same shape.

### Chat
| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/chat` | Send a message, receive the complete JSON response (message + executed actions) |
| GET | `/api/chat/history` | Last 50 messages, oldest first, to restore the chat panel on page load |

`POST /api/chat` with body `{"message": "Buy 5 shares of NVDA"}` → 200
```json
{
  "id": "uuid",
  "role": "assistant",
  "content": "Buying 5 NVDA at market.",
  "actions": {
    "trades": [
      {"ticker": "NVDA", "side": "buy", "quantity": 5, "status": "executed", "price": 875.10, "error": null}
    ],
    "watchlist_changes": [
      {"ticker": "PYPL", "action": "add", "status": "failed", "error": "Invalid ticker"}
    ]
  },
  "created_at": "2026-10-22T14:30:02Z"
}
```
`status` is `"executed"` or `"failed"`; on failure, `price` is null and `error` holds the reason. If the LLM call itself fails or returns unparseable output, the response is still 200, with a friendly `content` such as "Sorry, I couldn't process that — please try again", and empty action lists.

`GET /api/chat/history` → 200: an array of objects shaped like the `POST /api/chat` response, with `role` either `"user"` or `"assistant"` (`actions` is null for user messages).

### System
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/health` | `{"status": "ok"}` (for Docker/deployment) |

### Trade Execution Rules

These rules are shared by manual trades and LLM trades:

1. Normalize and validate the ticker; `quantity` must be ≥ 0.0001 (fractional allowed); `side` ∈ {`buy`, `sell`}.
2. If the ticker isn't tracked, add it to the watchlist (which starts tracking it). In simulator mode a price is available immediately; with Massive the trade fails with "Price unavailable" until the next poll.
3. Fill price = the current cached price. If none, reject.
4. **Buy:** require `cash ≥ qty × price`. New `avg_cost = (old_qty × old_avg + qty × price) / (old_qty + qty)`.
5. **Sell:** require `held_qty ≥ qty` (allowing a tolerance of 1e-9). `avg_cost` is unchanged. If the remaining quantity is ≤ 1e-9, delete the position row and untrack the ticker if it isn't on the watchlist.
6. In **one DB transaction**: update cash, upsert or delete the position, insert the trade, and insert a portfolio snapshot.
7. Serialize trade execution with a single `asyncio.Lock`, so concurrent manual and LLM trades can't race.

Realized P&L is **not** tracked or displayed. It's derivable later from the `trades` log if wanted.

---

## 9. LLM Integration

When writing code to make calls to LLMs, use cerebras-inference skill to use LiteLLM via OpenRouter to the `openrouter/openai/gpt-oss-120b` model with Cerebras as the inference provider. Structured Outputs should be used to interpret the results.

There is an OPENROUTER_API_KEY in the .env file in the project root.

> **IMPORTANT — no streaming.** Make one LLM call per chat message and return one complete JSON payload. Structured output has to arrive whole before it can be parsed, and Cerebras responds in 1–3 seconds, so token streaming would add complexity with no real UX benefit. The frontend shows a loading indicator while it waits.

### How It Works

When the user sends a chat message, the backend:

1. Stores the user message in `chat_messages`
2. Loads the user's current portfolio context (cash, positions with P&L, watchlist with live prices, total portfolio value)
3. Loads the last 20 messages from `chat_messages` as conversation history. For assistant messages, include a compact summary of their `actions`, so the LLM knows what succeeded or failed previously.
4. Constructs a prompt with a system message, portfolio context, conversation history, and the user's new message
5. Calls the LLM via LiteLLM → OpenRouter with structured output, using the cerebras-inference skill
6. Parses and validates the complete JSON response against the schema
7. Executes trades **sequentially and independently**, in order: one failing trade doesn't stop or roll back the others. Then applies watchlist changes the same way. Trades follow the rules in §8.
8. Stores the assistant message together with its `actions` (including per-item status/errors)
9. Returns the response described in §8 (`POST /api/chat`)

### Structured Output Schema

The LLM must respond with JSON matching this schema. All fields are **required**; the arrays are empty when unused (strict-mode friendly):

```json
{
  "message": "Your conversational response to the user",
  "trades": [
    {"ticker": "AAPL", "side": "buy", "quantity": 10}
  ],
  "watchlist_changes": [
    {"ticker": "PYPL", "action": "add"}
  ]
}
```

- `message` (string): The conversational text shown to the user
- `trades` (array): Trades to auto-execute. `side` is an enum: `"buy"` | `"sell"`. `quantity` is a number > 0.
- `watchlist_changes` (array): `action` is an enum: `"add"` | `"remove"`

Define the schema as a Pydantic model and pass it as the `response_format`. If parsing or validation fails, execute nothing and return the friendly fallback message (§8).

### Auto-Execution

Trades specified by the LLM execute automatically — no confirmation dialog. This is a deliberate design choice:
- It's a simulated environment with fake money, so the stakes are zero
- It creates an impressive, fluid demo experience
- It demonstrates agentic AI capabilities — the core theme of the course

The LLM writes its message *before* trades are executed, so it can't comment on the outcome in the same turn. Failed actions are instead shown inline by the frontend, using the per-item `status`/`error` in `actions`. The LLM sees those outcomes on the next turn through conversation history. The system prompt should tell the LLM to phrase trade messages as intent ("Buying 5 NVDA…") rather than confirmation.

### System Prompt Guidance

The LLM should be prompted as "FinAlly, an AI trading assistant" with instructions to:
- Analyze portfolio composition, risk concentration, and P&L
- Suggest trades with reasoning
- Execute trades when the user asks or agrees
- Manage the watchlist proactively
- Be concise and data-driven in responses
- Always respond with valid structured JSON

### LLM Mock Mode

When `LLM_MOCK=true`, the backend skips the LLM call and builds the structured response deterministically from the user's message. The first rule that matches (case-insensitive) wins:

| User message contains | Mock response |
|---|---|
| `buy` | `message: "Mock: buying 1 share of AAPL."`, `trades: [{"ticker": "AAPL", "side": "buy", "quantity": 1}]` |
| `sell` | `message: "Mock: selling 1 share of AAPL."`, `trades: [{"ticker": "AAPL", "side": "sell", "quantity": 1}]` |
| `watch` | `message: "Mock: adding PYPL to your watchlist."`, `watchlist_changes: [{"ticker": "PYPL", "action": "add"}]` |
| anything else | `message: "Mock: I'm FinAlly, your AI trading assistant."`, no actions |

The mock response then goes through the normal execution, storage, and response steps, so E2E tests exercise the real flow. This enables:
- Fast, free, reproducible E2E tests
- Development without an API key
- CI/CD pipelines

---

## 10. Frontend Design

### Layout

The frontend is a single-page application with a dense, terminal-inspired layout. The specific component architecture and layout system is up to the Frontend Engineer, but the UI should include these elements:

- **Watchlist panel** — grid/table of watched tickers with: ticker symbol, current price (flashing green/red on change), daily change % (`day_change_percent` from SSE), a sparkline mini-chart (accumulated from SSE since page load), and a remove control. Tickers without a price yet show "—".
- **Main chart area** — larger chart for the currently selected ticker, showing price over time since page load. Clicking a ticker in the watchlist selects it here.
- **Portfolio heatmap** — treemap visualization where each rectangle is a position, sized by portfolio weight, colored by P&L (green = profit, red = loss)
- **P&L chart** — line chart showing total portfolio value over time, using `GET /api/portfolio/history`
- **Positions table** — tabular view of all positions: ticker, quantity, avg cost, current price, unrealized P&L, % change
- **Trade bar** — simple input area: ticker field, quantity field, buy button, sell button. Market orders, instant fill. Errors (`detail`) shown inline.
- **AI chat panel** — docked/collapsible sidebar. History loaded from `GET /api/chat/history` on page load. Message input, scrolling conversation history, loading indicator while waiting for LLM response. Each action is shown inline as a confirmation (executed) or as an error (failed).
- **Header** — portfolio total value (updating live), connection status indicator, cash balance

### Data Flow

- **Prices:** a single `EventSource` to `/api/stream/prices` feeds a per-ticker in-memory ring buffer (last ~500 points). Sparklines and the main chart both read this buffer. The main chart shows data **only since page load**. There is intentionally **no** backend price-history endpoint, so don't build one.
- **Portfolio:** fetch `GET /api/portfolio` on load and after every trade or chat response that contains actions. On every SSE event, the frontend recomputes the live values: each position's `current_price`, `market_value`, and P&L, plus header `total_value = cash + Σ quantity × live price`.
- **P&L chart:** fetch `GET /api/portfolio/history` on load and after trades. Re-poll every 30s.
- **Watchlist:** fetch `GET /api/watchlist` on load and after any watchlist change (manual or via chat). Live prices come from SSE.
- **Connection status:** green when `EventSource` is open, yellow while it's reconnecting (an error fired but `readyState` is `CONNECTING`), red when it's `CLOSED`.

### Technical Notes

- **Charting:** use **Lightweight Charts** (TradingView, canvas) for the main price chart and the P&L chart. Draw sparklines as small inline SVG polylines (no library). Build the heatmap from `d3-hierarchy`'s `treemap()` layout, rendered as absolutely positioned divs. Don't evaluate other libraries.
- Price flash effect: on receiving a new price, briefly apply a CSS class with background color transition, then remove it
- All API calls go to the same origin (`/api/*`) — no CORS configuration needed
- Tailwind CSS for styling with a custom dark theme
- `next.config`: `output: 'export'` and `images: { unoptimized: true }`

---

## 11. Docker & Deployment

### Multi-Stage Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - npm ci && npm run build (produces static export in out/)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/ to /app
  - uv sync --frozen (install Python dependencies from lockfile)
  - Copy frontend out/ to /app/static
  - ENV DB_PATH=/app/db/finally.db
  - Expose port 8000
  - CMD: uvicorn app.main:app --host 0.0.0.0 --port 8000
```

FastAPI serves all API routes and the static frontend on port 8000. Register all `/api` routers **first**, then mount `StaticFiles(directory="static", html=True)` at `/`, so the static catch-all never shadows the API. Skip the mount if `static/` doesn't exist, for local backend-only runs.

### Docker Volume

The SQLite database persists in a **named Docker volume** (portable, and avoids Windows bind-mount path issues):

```bash
docker run -d --name finally -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

The top-level `db/` directory is used only for local (non-Docker) runs.

### Start/Stop Scripts

**`scripts/start_mac.sh`** (macOS/Linux):
- Builds the Docker image if not already built (or if `--build` flag passed)
- Removes any existing stopped `finally` container, then runs the container with the volume, port mapping, and `.env` file
- Prints the URL to access the app
- Optionally opens the browser

**`scripts/stop_mac.sh`** (macOS/Linux):
- Stops and removes the running container
- Does NOT remove the volume (data persists)

**`scripts/start_windows.ps1`** / **`scripts/stop_windows.ps1`**: PowerShell equivalents for Windows.

All scripts should be idempotent — safe to run multiple times.

### Optional Cloud Deployment

The container is designed to deploy to AWS App Runner, Render, or any container platform. A Terraform configuration for App Runner may be provided in a `deploy/` directory as a stretch goal, but is not part of the core build.

---

## 12. Testing Strategy

### Unit Tests (within `frontend/` and `backend/`)

**Backend (pytest)**:
- Market data: complete (73 tests). Add tests for `open_price` / `day_change_percent`.
- Portfolio: trade execution rules (§8), including the weighted average cost, a sell-all that deletes the position, selling more than owned, buying with insufficient cash, selling at a loss, no-price rejection, and auto-tracking of unwatched tickers
- LLM: schema parsing for valid responses, fallback on malformed responses, sequential and independent execution with mixed success and failure, and each mock-mode rule
- API routes: status codes, response shapes matching §8, and the error format

**Frontend (React Testing Library or similar)** — keep it lean:
- Portfolio calculation helpers (live total value, P&L)
- Connection-status state logic (open / reconnecting / closed)
- Render smoke tests for the watchlist, positions table, and chat (including a failed action and the loading state)

### E2E Tests (in `test/`)

**Infrastructure**: A separate `docker-compose.test.yml` in `test/` that spins up the app container plus a Playwright container. This keeps browser dependencies out of the production image.

**Environment**: Tests run with `LLM_MOCK=true` and a fresh volume for speed and determinism.

**Key Scenarios**:
- Fresh start: default watchlist appears, $10k balance shown, prices are streaming
- Add and remove a ticker from the watchlist
- Buy shares: cash decreases, position appears, portfolio updates
- Sell shares: cash increases, position updates or disappears
- Portfolio visualization: heatmap renders with correct colors, P&L chart has data points
- AI chat (mocked): send "buy", receive a response, the executed trade appears inline and the position appears; reload the page and chat history is still shown

---

## 13. Review Decision Log

The 2026-10-22 review items were resolved and incorporated into the sections above. IDs are kept so agents can refer to them.

| ID | Decision | Where |
|---|---|---|
| R1 | Full request/response JSON shapes and the error convention defined | §8 |
| R2 | SSE payload documented: one event per change, holding a map of all tickers | §6 |
| R3 | Chat history persists; `GET /api/chat/history` added | §8, §10 |
| R4 | Mock LLM is keyword-driven and runs through the real execution flow | §9 |
| R5 | "Daily change" baseline = `open_price` (first price since server start); small addition to market code | §6 |
| R6 | Tracked tickers = watchlist ∪ positions; trading an unwatched ticker auto-adds it; no price → reject | §6, §8 |
| R7 | Any ticker matching `^[A-Z.]{1,10}$` is accepted in simulator mode | §8 |
| R8 | Trade rules spelled out; realized P&L not tracked | §8 |
| R9 | Single LLM call; failures reported per action and shown by the UI | §9 |
| R10 | All schema fields required, enums for `side`/`action`, fallback on parse failure | §9 |
| R11 | Last 20 messages sent as history | §9 |
| R12 | LLM trades run sequentially and independently | §9 |
| R13 | Frontend recomputes live total value from SSE | §10 |
| R14 | `users_profile` exception to the `user_id` rule | §7 |
| R15 | Named volume only | §11 |
| R16 | `--env-file` in Docker, `python-dotenv` locally; `DB_PATH` added | §5 |
| R17 | Lightweight Charts + SVG sparklines + d3-hierarchy treemap | §10 |
| R18 | DB code in `backend/app/db/` | §4 |
| R19 | DB initialized at startup only | §7 |
| — | No LLM streaming (owner comment) | §3, §9 |
| S1 | Root `docker-compose.yml` dropped | §4 |
| S2 | Shared frontend price buffer; no backend price history | §10 |
| S3 | History endpoint capped at 500 points | §8 |
| S4 | `DEFAULT_USER_ID` constant, no multi-user plumbing | §7 |
| S5 | Leaner frontend tests; SSE-resilience E2E replaced by a connection-status unit test | §12 |
| S6 | Static mount order and `next.config` settings | §10, §11 |
| S7 | Rounding at the API boundary | §8 |
