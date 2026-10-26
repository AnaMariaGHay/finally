---
last_mapped_commit: b97d1fef40a1301a65818a1554fd2d67cc274d27
last_mapped_at: 2026-10-26
---
<!-- refreshed: 2026-10-26 -->

# Architecture

**Analysis Date:** 2026-10-26

## System Overview

FinAlly is a single-container AI trading workstation. Currently, the **market data subsystem is complete and tested**; the **API layer, database layer, LLM integration, and frontend remain to be built**. This document covers both what exists and the planned architecture.

```text
┌─────────────────────────────────────────────────────────────┐
│                      Frontend Layer                         │
│            (Next.js → Static Export) [PLANNED]              │
│                                                             │
│  ├── Watchlist Panel        ├── Chart Area                 │
│  ├── Portfolio Heatmap      ├── Positions Table            │
│  ├── P&L Chart              └── AI Chat Panel              │
└──────────────────────┬──────────────────────────────────────┘
                       │ /api/* (REST) + /api/stream/* (SSE)
┌──────────────────────▼──────────────────────────────────────┐
│                    FastAPI Backend                          │
│         `backend/app/` with `backend/pyproject.toml`        │
│                                                             │
│  ├─ API Layer (Routes) [PLANNED]                           │
│  │  ├── /api/portfolio (GET, POST /trade)                  │
│  │  ├── /api/watchlist (GET, POST, DELETE)                 │
│  │  ├── /api/chat (POST, LLM integration)                  │
│  │  └── /api/health                                        │
│  │                                                         │
│  ├─ Market Data Layer [COMPLETE]                           │
│  │  └── `app/market/` (8 modules, 500 lines)               │
│  │      ├── Price Cache (thread-safe, in-memory)           │
│  │      ├── Market Data Sources (Simulator + Massive API)  │
│  │      └── SSE Stream Router (/api/stream/prices)         │
│  │                                                         │
│  ├─ Database Layer [PLANNED]                               │
│  │  ├── `app/db/` schema, seed data, ORM/queries           │
│  │  └── SQLite → `db/finally.db` (volume-mounted)          │
│  │                                                         │
│  └─ LLM Integration [PLANNED]                              │
│     ├── OpenRouter client via LiteLLM                       │
│     └── Structured output parsing + auto-execution         │
│                                                             │
│  ├─ Background Tasks [PARTIALLY COMPLETE]                  │
│  │  ├── Market data source (Simulator or Massive)          │
│  │  └── Portfolio snapshots (every 30s, post-trade)        │
│  │                                                         │
│  └─ Static File Serving [PLANNED]                          │
│     └── Next.js export output from `frontend/out/`         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              SQLite Database (single file)                  │
│               `db/finally.db` [PLANNED]                     │
│                                                             │
│  ├── users_profile          ├── trades                      │
│  ├── watchlist              ├── portfolio_snapshots         │
│  ├── positions              └── chat_messages               │
└─────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File(s) | Status |
|-----------|----------------|---------|--------|
| PriceCache | Thread-safe in-memory store of latest prices | `app/market/cache.py` | Complete |
| MarketDataSource | Abstract interface for price data providers | `app/market/interface.py` | Complete |
| SimulatorDataSource | GBM-based price simulator (default) | `app/market/simulator.py` | Complete |
| MassiveDataSource | Polygon.io REST API client (real data) | `app/market/massive_client.py` | Complete |
| SSE Stream Router | Endpoint that pushes prices to clients | `app/market/stream.py` | Complete |
| API Routes | REST endpoints for portfolio, watchlist, chat | `app/routes/` or similar | Planned |
| Database | SQLite schema, migrations, seed data | `app/db/` | Planned |
| LLM Client | OpenRouter integration via LiteLLM | `app/llm/` | Planned |

## Pattern Overview

**Overall:** Layered architecture with dependency injection, factory pattern for component creation, and interface-based abstraction for swappable implementations.

**Key Characteristics:**
- Single-container deployment (FastAPI + static frontend + SQLite)
- Environment-variable driven behavior (MASSIVE_API_KEY selects market source, OPENROUTER_API_KEY enables chat)
- Async/await throughout backend for concurrent request handling
- Thread-safe shared cache for price updates (multiple SSE clients read, one producer writes)
- Structured outputs for LLM → JSON with trade/watchlist actions, auto-executed immediately

## Layers

**Presentation Layer (Frontend):**
- Purpose: Browser UI for watching prices, trading, and chatting with AI
- Location: `frontend/` (not yet created; will build static export)
- Contains: React/TypeScript components, Tailwind styling, EventSource SSE client
- Depends on: `/api/*` and `/api/stream/*` endpoints
- Used by: End user's browser

**API Layer (Planned):**
- Purpose: HTTP endpoints for all user actions and data queries
- Location: `app/routes/` or top-level FastAPI file
- Contains: Portfolio, watchlist, chat, health endpoints with validation and error handling
- Depends on: Database layer, LLM client, market data cache
- Used by: Frontend client

**Market Data Layer (Complete):**
- Purpose: Provide real-time ticker prices via single-source-of-truth cache
- Location: `app/market/`
- Contains: Price models, cache, data source interface, simulator, Massive client, SSE streamer
- Depends on: Environment variables, optional Massive API
- Used by: Portfolio valuation, trade execution, SSE streaming, API routes

**Database Layer (Planned):**
- Purpose: Persistent storage of user state (positions, trades, chat history)
- Location: `app/db/`
- Contains: SQLite schema, seed data, ORM query builders or raw SQL
- Depends on: SQLite file at `db/finally.db`
- Used by: Portfolio routes, trade execution, watchlist, chat history

**LLM Integration (Planned):**
- Purpose: AI chat assistant with portfolio analysis and auto-trade execution
- Location: `app/llm/` or top-level chat route
- Contains: LiteLLM client, structured output parser, trade/watchlist action executor
- Depends on: OPENROUTER_API_KEY, database for portfolio context, market data for prices
- Used by: Chat endpoint

## Data Flow

### Primary Request Path: Stream Prices (Complete)

1. **Client connects** — browser opens EventSource to `/api/stream/prices` (`app/market/stream.py:stream_prices`)
2. **Event generator spawns** — `_generate_events()` starts yielding SSE events at 500ms intervals
3. **Price fetch** — reads from `PriceCache.get_all()` (`app/market/cache.py`)
4. **Market data producer writes** — `SimulatorDataSource` or `MassiveDataSource` calls `cache.update(ticker, price)` periodically
5. **Version check** — SSE generator compares `cache.version` to detect changes, only sends if updated
6. **Event output** — JSON payload with all tickers' latest prices sent as `data: {...}\n\n`
7. **Client receives** — browser's EventSource listener parses JSON, updates watchlist prices, triggers flash animation

### Trade Execution (Planned)

1. **User submits trade** — POST `/api/portfolio/trade` with `{ticker, quantity, side}` (buy/sell)
2. **Get current price** — `cache.get_price(ticker)` from market data
3. **Validate** — check cash balance (buy) or position quantity (sell) against database
4. **Execute** — deduct/add cash, create/update position, log trade to `trades` table
5. **Record snapshot** — append portfolio value to `portfolio_snapshots` table
6. **Return confirmation** — new portfolio state to frontend

### Chat with AI (Planned)

1. **User sends message** — POST `/api/chat` with `{message}`
2. **Load context** — fetch portfolio (positions, cash, P&L), watchlist with prices, recent chat history from database
3. **LLM call** — POST to OpenRouter via LiteLLM with system prompt + context + history + user message
4. **Parse response** — structured output JSON with `message`, `trades[]`, `watchlist_changes[]`
5. **Auto-execute** — execute each trade (same flow as manual trade), apply watchlist changes
6. **Store** — save user message and AI response + executed actions to `chat_messages` table
7. **Return** — JSON response with conversational message and action confirmations

**State Management:**
- **Prices**: PriceCache (in-memory, authoritative source during runtime)
- **Positions & Trades**: SQLite database (durable, single source of truth)
- **Session state**: Frontend stores accumulated sparkline data in memory; SSE client reconnects on disconnect
- **Portfolio snapshots**: Background task saves to database every 30s and after each trade

## Key Abstractions

**MarketDataSource Interface:**
- Purpose: Define contract for price providers (simulator, real API)
- Implementations: `SimulatorDataSource`, `MassiveDataSource`
- Pattern: Strategy pattern — swappable at runtime via factory
- Files: `app/market/interface.py`, `app/market/factory.py`

**PriceUpdate Dataclass:**
- Purpose: Immutable snapshot of a ticker's price with computed metrics
- Properties: `change`, `change_percent`, `direction`, `to_dict()`
- Why immutable: Thread-safe for concurrent reads by SSE clients and API routes
- File: `app/market/models.py`

**PriceCache:**
- Purpose: Single point of truth for all ticker prices
- Design: Thread-safe via Lock, version counter for change detection (SSE optimization)
- Writers: Single market data source (simulator or Massive poller)
- Readers: SSE stream, portfolio routes, trade execution, LLM context loading
- File: `app/market/cache.py`

**SSE Stream Router (Planned Extension):**
- Purpose: FastAPI APIRouter factory that creates the `/api/stream/prices` endpoint
- Injection: Receives `price_cache` as parameter, no global state
- Resilience: EventSource auto-reconnects on disconnect; server sends `retry: 1000` directive
- File: `app/market/stream.py`

## Entry Points

**Frontend Entry Point (Planned):**
- Location: `frontend/pages/index.tsx` or `frontend/app/page.tsx`
- Triggers: Browser loads `http://localhost:8000/`
- Responsibilities: Initialize SSE connection, render dashboard, set up component interactivity

**Backend Entry Point (Planned):**
- Location: `backend/app/main.py` or similar (created during API phase)
- Triggers: uvicorn loads the FastAPI app on container startup
- Responsibilities: Create FastAPI instance, register routers, initialize database, start background tasks (market data, snapshots)

**Market Data Background Task (Complete):**
- Location: `app/market/simulator.py:SimulatorDataSource.start()` or `massive_client.py:MassiveDataSource.start()`
- Triggers: Called during app startup
- Responsibilities: Run background coroutine that updates prices in the cache every 500ms (simulator) or ~10s (Massive)

**SSE Stream Endpoint (Complete):**
- Location: `app/market/stream.py:stream_prices()` (FastAPI route)
- Triggers: GET `/api/stream/prices`
- Responsibilities: Accept SSE client connection, yield price updates until disconnection

## Architectural Constraints

- **Threading:** Python's GIL is mitigated by async/await; market data source and SSE generator run in asyncio tasks, no threads needed for concurrency. PriceCache uses Lock for thread-safe access from multiple coroutines.
- **Global state:** None by design. Each component receives dependencies via constructor (dependency injection). FastAPI app instance created in `main.py`, components access it via function parameters.
- **Circular imports:** Avoided via clear dependency direction: market data (models, cache, sources) have no dependencies on higher layers; API routes depend on market data; LLM integration depends on all lower layers.
- **Shared state:** Only PriceCache is intentionally shared (producer-consumer pattern). Database is also shared (all routes query it). Frontend state is local to each browser session.
- **Single container:** No inter-process communication; everything runs in one Python process. SQLite database is local file at `db/finally.db`.
- **SSE stream scaling:** Each client gets its own async generator; no shared connection state. Works fine for single-user development; multi-user would require broadcast channel or WebSocket.

## Anti-Patterns

### Global Market Data Source

**What happens:** Backend could keep a global `SOURCE` variable in `app/market/factory.py` to avoid passing the source around.

**Why it's wrong:** Testing becomes harder (can't inject a mock), and multiple instances or background tasks can't coexist. The plan calls for background market updates + SSE streaming, both needing the source.

**Do this instead:** Create the source during app startup in `main.py` and pass it (or the PriceCache) to all routes and tasks via function parameters or FastAPI dependency injection.

### Hardcoded SQL in Routes

**What happens:** API routes build SQL strings by hand without type safety or query builders.

**Why it's wrong:** SQL injection risk, typos in column names, no compile-time checking of schema changes.

**Do this instead:** Use an ORM like SQLAlchemy, Tortoise, or Pydantic with raw SQL, or query builder with type checking. File location: `app/db/models.py` or similar.

### Blocking Database Calls in Async Handlers

**What happens:** Routes call `sqlite3.connect()` synchronously in async functions, blocking the event loop.

**Why it's wrong:** Stalls all other concurrent requests. Market data updates won't be delivered while a trade query runs.

**Do this instead:** Use an async SQLite client like `aiosqlite`, or use a thread pool executor for blocking calls. See `backend/tests/market/test_*.py` for async test patterns.

### Storing Chat History in Memory

**What happens:** Chat messages accumulated in a list in the FastAPI app instance.

**Why it's wrong:** Lost on app restart; not shared if scaled to multiple processes; memory leak if chat runs long.

**Do this instead:** Persist to database. File location: `chat_messages` table in schema at `app/db/schema.sql` or equivalent.

## Error Handling

**Strategy:** Explicit is better than implicit. Errors are caught at boundaries (routes) and returned as HTTP responses with clear status codes.

**Patterns:**
- Market data source fails → PriceCache returns `None` on `.get()`, routes handle gracefully (show "unavailable" or cached value)
- Trade validation fails → return 400 Bad Request with error message
- Database unavailable → return 503 Service Unavailable
- LLM API fails → return 502 Bad Gateway; frontend shows error to user
- SSE client disconnects → generator catches exception, logs, stops (not an error, expected)

## Cross-Cutting Concerns

**Logging:** Use Python's `logging` module configured in `app/__init__.py` or `main.py`. Market data modules already use it (e.g., `app/market/simulator.py:logger.info()`). Apply same pattern to routes and database layer.

**Validation:** Route inputs validated via Pydantic models (FastAPI does this automatically). Database schema enforces constraints (NOT NULL, UNIQUE, CHECK). LLM inputs validated before sending to API.

**Authentication:** Not implemented (single-user by design). `user_id` hardcoded to `"default"` in all database queries.

**Structured Logging:** Consider JSON structured logs for production (tools like `python-json-logger`), but standard logs suffice for single-container deployment.

---

*Architecture analysis: 2026-10-26*
