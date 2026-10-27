# Requirements: FinAlly — AI Trading Workstation

**Defined:** 2026-10-27
**Core Value:** A user runs one Docker command, opens `localhost:8000`, and immediately sees live-streaming prices, can buy/sell instantly, and can ask the AI copilot to analyze and trade for them — all working end-to-end in a single container.

Source spec: `planning/PLAN.md`. Research: `.planning/research/SUMMARY.md`.

## v1 Requirements

### Platform (backend foundation)

- [ ] **PLAT-01**: Backend starts as a single FastAPI app on port 8000 (one uvicorn worker) that wires the market data source, SSE stream, API routers, and background tasks via an app factory + lifespan
- [ ] **PLAT-02**: On first start with no/empty `db/finally.db`, the backend creates all six tables (users_profile, watchlist, positions, trades, portfolio_snapshots, chat_messages, each with `user_id` default `"default"`) and seeds $10,000 cash plus the 10 default tickers; existing data is preserved on restart
- [ ] **PLAT-03**: Schema/seed logic lives in the Python package (`backend/app/db/`) and the DB path is configurable and absolute so the Docker volume at `/app/db` never shadows code
- [ ] **PLAT-04**: `GET /api/health` returns 200 with a simple status payload
- [ ] **PLAT-05**: The existing SSE router can be created more than once without duplicate routes, and SSE generators shut down cleanly on server stop
- [ ] **PLAT-06**: Database access never blocks the event loop (SSE keeps streaming during trades and chat calls)
- [ ] **PLAT-07**: FastAPI serves the Next.js static export at `/` with all `/api/*` routes taking precedence; backend still starts if the static directory is absent

### Market Data Extensions

- [ ] **MKT-01**: Each price carries a daily baseline — session-open price for the simulator, previous close for Massive — and a `day_change_percent` exposed via SSE and `/api/watchlist`
- [ ] **MKT-02**: The set of tickers being priced is always watchlist ∪ held positions, so removing a watched ticker that is still held keeps it priced
- [ ] **MKT-03**: Tickers are validated as 1–5 letter uppercase symbols (normalized to uppercase) before being added or traded

### Watchlist

- [ ] **WATCH-01**: User can fetch the watchlist with each ticker's latest price, daily change %, and baseline
- [ ] **WATCH-02**: User can add a valid ticker; it starts streaming prices; duplicates are rejected/idempotent with a clear response
- [ ] **WATCH-03**: User can remove a ticker; it stops streaming unless held in a position
- [ ] **WATCH-04**: Invalid tickers return 400 with a readable error

### Portfolio & Trading

- [ ] **PORT-01**: User can fetch portfolio: cash, positions (ticker, quantity, avg cost, current price, unrealized P&L, unrealized % vs avg cost), total value, total unrealized P&L
- [ ] **PORT-02**: User can buy a market order `{ticker, quantity, side:"buy"}` that fills instantly at the current cached price; fractional quantities supported
- [ ] **PORT-03**: User can sell a market order that fills instantly; avg cost unchanged on sells; position removed when quantity reaches ~0
- [ ] **PORT-04**: Buys with insufficient cash, sells exceeding held quantity, non-positive/NaN quantities, and tickers with no price yet are rejected with 400 and a specific message; no partial state is written
- [ ] **PORT-05**: Each trade (cash, position, trade log, snapshot) commits atomically; concurrent trades (manual + AI) cannot corrupt cash or positions
- [ ] **PORT-06**: Trading a ticker not on the watchlist auto-adds it to the watchlist
- [ ] **PORT-07**: Every trade is appended to the `trades` log
- [ ] **PORT-08**: A portfolio snapshot is recorded every 30 seconds by a background task and immediately after each trade
- [ ] **PORT-09**: User can fetch portfolio value history (`GET /api/portfolio/history`) for the P&L chart

### AI Chat

- [ ] **CHAT-01**: User can send a message to `POST /api/chat` and receive a complete JSON response `{message, actions}` (no streaming)
- [ ] **CHAT-02**: The LLM prompt includes the system persona ("FinAlly, an AI trading assistant"), current cash, positions with P&L, watchlist with live prices, total value, and recent conversation history
- [ ] **CHAT-03**: The LLM is called via LiteLLM → OpenRouter `openrouter/openai/gpt-oss-120b` with Cerebras provider and structured output `{message, trades[], watchlist_changes[]}`, asynchronously (non-blocking)
- [ ] **CHAT-04**: Trades and watchlist changes from the LLM auto-execute (watchlist changes first, then trades) through the same validation as manual actions; each returns a per-action result (executed with price, or failed with error)
- [ ] **CHAT-05**: Malformed/empty LLM output or LLM API failures produce a graceful assistant error message instead of a 500 crash
- [ ] **CHAT-06**: User and assistant messages (with executed actions JSON) are persisted in `chat_messages`
- [ ] **CHAT-07**: User can fetch recent chat history (`GET /api/chat`) so the conversation survives page reload
- [ ] **CHAT-08**: With `LLM_MOCK=true`, chat returns deterministic mock responses (including a mock trade path) without importing/calling the real LLM or needing an API key

### Frontend UI

- [ ] **UI-01**: Single-page Next.js TypeScript app built as a static export, Tailwind dark theme (`#0d1117`-ish backgrounds, muted borders, accent `#ecad0a`, blue `#209dd7`, purple `#753991` submit buttons), desktop-first and functional on tablet
- [ ] **UI-02**: Header shows live total portfolio value, cash balance, and a connection status dot (green connected / yellow reconnecting / red disconnected)
- [ ] **UI-03**: A single shared EventSource to `/api/stream/prices` feeds a central price store; it recovers automatically after disconnects (including manual re-create when the browser gives up)
- [ ] **UI-04**: Watchlist panel lists each ticker with current price, daily change %, and a sparkline accumulated from SSE since page load
- [ ] **UI-05**: Prices flash green (uptick) or red (downtick) on change, fading over ~500ms via CSS transition; no flash on initial load
- [ ] **UI-06**: User can add and remove tickers from the watchlist panel
- [ ] **UI-07**: Clicking a watchlist ticker shows a larger live price chart for it in the main chart area
- [ ] **UI-08**: Portfolio heatmap (treemap) shows positions sized by portfolio weight and colored green/red by P&L, with an empty state when no positions
- [ ] **UI-09**: P&L line chart shows total portfolio value over time from snapshots
- [ ] **UI-10**: Positions table shows ticker, quantity, avg cost, current price, unrealized P&L, % change, updating live
- [ ] **UI-11**: Trade bar with ticker and quantity fields plus Buy and Sell buttons executes instantly (no confirmation dialog) and shows success/error feedback
- [ ] **UI-12**: Docked, collapsible AI chat panel with scrolling history (loaded from server), message input, loading indicator while waiting, and inline confirmation chips for executed/failed trades and watchlist changes
- [ ] **UI-13**: After any manual trade or AI action, portfolio, positions, watchlist, and history views refresh

### Packaging & Scripts

- [ ] **PKG-01**: Multi-stage Dockerfile (Node build stage → Python 3.12 + uv runtime) produces one image that serves the full app on port 8000
- [ ] **PKG-02**: `docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally` persists the SQLite DB across container restarts
- [ ] **PKG-03**: Idempotent `scripts/start_mac.sh` / `stop_mac.sh` (build-if-needed or `--build`, run with volume/port/env, print URL, optionally open browser; stop without removing volume)
- [ ] **PKG-04**: Idempotent `scripts/start_windows.ps1` / `stop_windows.ps1` PowerShell equivalents
- [ ] **PKG-05**: Repo hygiene for cross-platform builds: `.gitattributes` (LF for shell scripts), `.dockerignore`, `.env.example`, `db/.gitkeep`, optional `docker-compose.yml`

### Testing

- [ ] **TEST-01**: Backend pytest covers trade execution and P&L math, edge cases (oversell, insufficient cash, selling at a loss, fractional), atomic/concurrent trades, and snapshots
- [ ] **TEST-02**: Backend pytest covers LLM structured-output parsing (valid schemas, malformed responses), chat action execution/validation, and mock mode
- [ ] **TEST-03**: Backend pytest covers API routes: status codes, response shapes, error handling; existing 73 market tests keep passing
- [ ] **TEST-04**: Frontend unit tests cover component rendering with mock data, price flash triggering, watchlist add/remove, portfolio calculations, chat rendering and loading state
- [ ] **TEST-05**: `test/docker-compose.test.yml` spins up the app container (`LLM_MOCK=true`) plus a Playwright container
- [ ] **TEST-06**: E2E scenarios pass: fresh start (10 tickers, $10k, prices streaming); add/remove watchlist ticker; buy (cash down, position appears); sell (cash up, position updates/disappears); heatmap and P&L chart render; mocked AI chat with inline trade execution; SSE disconnect and reconnect

## v2 Requirements

Deferred; tracked but not in current roadmap.

- **V2-01**: Startup snapshot so the P&L chart has a point before the first trade/30s tick
- **V2-02**: Gated test-only DB reset endpoint for E2E isolation
- **V2-03**: Starter prompt chips in the empty chat panel
- **V2-04**: Average-cost price line on the main chart
- **V2-05**: Trade history view
- **V2-06**: Candlestick aggregation from ticks
- **V2-07**: Cloud deployment (Terraform / App Runner in `deploy/`)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Authentication / multi-user | Single-user by design; `user_id="default"` preserves future path |
| Limit/stop orders, order book, partial fills, fees | Market orders only — keeps portfolio math simple |
| Short selling, margin | Not in spec; sells capped at held quantity |
| Trade confirmation dialogs | Deliberate agentic demo choice; fake money |
| Token-by-token LLM streaming | Cerebras is fast; a loading indicator suffices |
| WebSockets | SSE one-way push is sufficient |
| Postgres / DB server | SQLite is zero-config for one user |
| Historical price backfill | Charts accumulate from SSE since page load |
| Timeframe selectors, indicators, news, fundamentals | Beyond spec; dense terminal look without them |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| (filled by roadmapper) | | |

**Coverage:**
- v1 requirements: 55 total
- Mapped to phases: 0
- Unmapped: 55 ⚠️

---
*Requirements defined: 2026-10-27*
*Last updated: 2026-10-27 after initial definition*
