# Architecture Research

**Domain:** AI-powered simulated trading workstation (FastAPI single container: REST + SSE + static Next.js export + SQLite + LLM)
**Researched:** 2026-10-27
**Confidence:** HIGH for everything derived from the existing `backend/app/market/` code (read directly); MEDIUM for Next.js-export/EventSource/LiteLLM/Lightweight Charts behaviors (from experience, not re-verified against current docs this session; flagged inline)

## Findings From The Existing Market Code (these drive the design)

| Fact (file) | Consequence for remaining work |
|---|---|
| `PriceCache` is a plain object with a `threading.Lock`; `source` and cache are constructed without an event loop (`SimulatorDataSource.__init__` / `MassiveDataSource.__init__` only store fields). `start()` is what spawns the task. | Cache **and** source can be created eagerly inside `create_app()`; only `start()/stop()` belong in `lifespan`. Router factories that need the cache at construction time (`create_stream_router`) work without hacks. |
| `create_stream_router` registers `@router.get("/prices")` on a **module-level** `router = APIRouter(...)` (`stream.py:17`). | Calling `create_app()` twice in one process (every test that builds an app) registers `/api/stream/prices` twice on the same router and the second closure never wins. **Fix: move `router = APIRouter(...)` inside the factory** (2-line change, no test impact). |
| SSE payload is **one unnamed `data:` event** containing a dict keyed by ticker: `{"AAPL": {ticker, price, previous_price, timestamp, change, change_percent, direction}, ...}`. First frame is `retry: 1000`. Event is emitted only when `cache.version` changed, checked every 500 ms. | Frontend uses `onmessage` (not named events), parses a map. With the simulator every tick bumps the version, so an event arrives ~2x/s containing **all** tickers. With Massive (15 s poll) events arrive rarely - frontend must de-dupe history by `timestamp`. |
| `PriceUpdate.previous_price` / `change_percent` / `direction` are **tick-to-tick**, not daily. First update of a ticker has `previous_price == price` (flat). | `change_percent` in the SSE payload is NOT the watchlist "daily change %". Direction is perfect for the flash animation; daily % needs a separate reference price (see "Session-open / reference price" below). |
| `MarketDataSource` exposes `add_ticker`, `remove_ticker`, `get_tickers()` (all idempotent, `add`/`remove` are `async`). `remove_ticker` also deletes the ticker from the cache. `SimulatorDataSource.add_ticker` seeds the cache immediately; `MassiveDataSource.add_ticker` does **not** - the price appears on the next poll (up to 15 s). `SimulatorDataSource` accepts *any* string (unknown tickers get `random.uniform(50, 300)`). | (1) Reconcile source tickers against DB using `get_tickers()` rather than bookkeeping increments. (2) Removing a watched ticker that has an open position would delete its price and break portfolio valuation - the tracked set must be `watchlist UNION positions`. (3) Ticker validation must be done by us (regex), and "no price yet" must be a first-class trade rejection. |
| `MassiveDataSource._poll_once` reads only `snap.last_trade.price/timestamp`; it ignores `prev_day`. | Previous-close for Massive requires a small extension in that one function (see below). |
| `to_dict()` tests assert individual keys, not dict equality (`test_models.py:59-70`). | Adding keys to `to_dict()` is safe. |
| `pyproject.toml` has `packages = ["app"]` for the wheel; deps lack litellm, pydantic (comes with fastapi), aiosqlite/none, python-dotenv, httpx (tests). | Schema/seed code must live **inside `app/`** (see structure note on `backend/db/` collision). Add `litellm` (+ `python-dotenv`, `httpx` dev) in the first phase that needs them. |

## Standard Architecture

### System Overview

```
 BROWSER (Next.js static export, all components 'use client')
 ┌──────────────────────────────────────────────────────────────────────────┐
 │  PriceStream (ONE EventSource, module singleton)                         │
 │       │ writes                                                           │
 │       ▼                                                                  │
 │  priceStore (Zustand, external mutable) ── latest{}, history{} ring bufs │
 │       │ selectors (per ticker)           ▲ connection status             │
 │       ▼                                  │                               │
 │  Watchlist+Sparklines | MainChart | Header | Heatmap | Positions | PnL   │
 │       ▲ derive live values = server positions x live prices              │
 │       │                                                                  │
 │  TanStack Query cache: ['portfolio'] ['watchlist'] ['history'] ['chat']  │
 │       ▲ invalidate on trade / watchlist change / chat with actions       │
 └───────┼──────────────────────────────────────────────────────────────────┘
         │ same origin: /api/* (JSON)  +  /api/stream/prices (SSE)
 ┌───────▼──────────────────────────────────────────────────────────────────┐
 │ FastAPI app  (create_app(), ONE uvicorn worker)                          │
 │                                                                          │
 │  ROUTES (thin; Pydantic in/out; map domain errors -> HTTP)               │
 │   health | portfolio | watchlist | chat | stream(existing) | static(last)│
 │       │            │             │                                       │
 │  SERVICES (framework-free, the only place business rules live)           │
 │   PortfolioService ◄── used by ──► ChatService (auto-exec) & REST        │
 │   WatchlistService ◄── used by ──► ChatService & REST                    │
 │   SnapshotService  ◄── used by ──► PortfolioService (post-trade) & loop  │
 │   LLMClient (real LiteLLM | Mock)                                        │
 │       │ reads/writes                  │ reads prices        │ add/remove │
 │       ▼                               ▼                     ▼            │
 │  db (sqlite3, WAL)               PriceCache ◄──writes── MarketDataSource │
 │  db/finally.db (volume)          (existing)             (Simulator|Massive)
 │                                                                          │
 │  lifespan: init db -> tracked tickers -> source.start -> snapshot task   │
 └──────────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Depends on | Status |
|-----------|----------------|-----------|--------|
| `PriceCache`, `MarketDataSource` impls, SSE router | Prices in, prices out | - | Exists (small extension: reference price; router fix) |
| `app/config.py` `Settings` | Read env once: `MASSIVE_API_KEY`, `OPENROUTER_API_KEY`, `LLM_MOCK`, `FINALLY_DB_PATH`, `FINALLY_STATIC_DIR` | - | New |
| `app/db/` (`schema.sql`, `connection.py`, `init_db()`) | Connection factory (WAL, busy_timeout, row factory), idempotent schema + seed | stdlib sqlite3 | New |
| `PortfolioService` (`app/portfolio/service.py`) | `get_portfolio()`, `execute_trade()` (atomic), `history()`; pure `valuation.py` helpers | db, PriceCache | New |
| `SnapshotService` (+ loop) | `record_snapshot()` immediately after trade and every 30 s | PortfolioService valuation, db | New |
| `WatchlistService` | add/remove with validation, **tracked-set reconcile** against source | db, MarketDataSource, PriceCache | New |
| `ChatService` (`app/chat/`) | Context build -> LLM -> parse -> execute -> persist | Portfolio/Watchlist services, LLMClient, db | New |
| `LLMClient` protocol: `LiteLLMClient`, `MockLLMClient` | One method `async complete(messages) -> ChatLLMOutput`; chosen once in `create_app()` | litellm | New |
| `AppContext` (dataclass on `app.state.ctx`) | Holds settings, cache, source, services; accessed via one `Depends(get_ctx)` | all | New |
| Static mount | Serves Next export; mounted **last** | filesystem | New |
| Frontend `priceStore`, `PriceStream`, Query hooks, panels | See "Frontend State Architecture" | - | New |

## Recommended Project Structure

```
backend/app/
├── main.py              # create_app(settings=None, *, source=None, llm=None, db_path=None); module-level `app = create_app()`
├── config.py            # Settings dataclass from env (+ dotenv load)
├── context.py           # AppContext dataclass + get_ctx dependency
├── market/              # EXISTING (extend cache.py/models.py/massive_client.py, fix stream.py)
├── db/
│   ├── schema.sql       # CREATE TABLE IF NOT EXISTS ... (6 tables)
│   ├── connection.py    # connect(path) -> sqlite3.Connection (WAL, FK, busy_timeout=5000)
│   └── init.py          # init_db(path): create + seed if profile row missing
├── portfolio/
│   ├── models.py        # Pydantic: TradeRequest, PositionOut, PortfolioOut, TradeResultOut
│   ├── valuation.py     # pure: position_view(), total_value(positions, cash, prices)
│   ├── service.py       # execute_trade(), get_portfolio(), get_history(); TradeError
│   ├── snapshots.py     # record_snapshot(), snapshot_loop()
│   └── routes.py
├── watchlist/
│   ├── service.py       # add/remove/list + reconcile_tracked()
│   └── routes.py
├── chat/
│   ├── schemas.py       # ChatLLMOutput (LLM-facing), ActionResult, ChatResponse
│   ├── prompts.py       # system prompt + context renderer
│   ├── llm.py           # LLMClient protocol, LiteLLMClient, parse_llm_output()
│   ├── mock.py          # MockLLMClient (deterministic keyword rules)
│   ├── service.py       # handle_message()
│   └── routes.py
└── static.py            # mount_frontend(app, dir) - no-op if dir missing
```

### Structure Rationale

- **Schema inside `app/db/`, NOT `backend/db/`:** PLAN.md section 4 says `backend/db/` holds schema SQL. But the runtime volume mount is `/app/db`; if the Dockerfile copies `backend/` to `/app/`, a `backend/db/` directory lands at `/app/db` and is **shadowed by the named volume**, so `schema.sql` vanishes in the container. `app/db/` is packaged with the wheel (`packages=["app"]`) and avoids the collision. Update the planning docs when this is done.
- **Services are framework-free (no FastAPI imports):** REST routes and ChatService both call the same functions. This is the single most important boundary - it guarantees "LLM trades go through the same validation as manual trades" by construction rather than by convention.
- **`create_app()` factory with injectable pieces:** tests pass a tmp-file DB, a `SimulatorDataSource` with a fast interval (or a fake source), and `MockLLMClient`. No globals (matches the existing codebase's DI stance).
- **Module-level `app = create_app()`** only in `main.py` for `uvicorn app.main:app`.

## Architectural Patterns

### Pattern 1: Single trade path (service shared by REST and LLM)

**What:** `PortfolioService.execute_trade(ticker, side, quantity) -> TradeResult` raises `TradeError(code, message)` for expected failures. REST maps to HTTP 400/404/422; ChatService catches and records `{status:"failed", error: message}`.
**When:** Always. Never duplicate validation in the chat layer.
**Trade-offs:** Slightly more indirection; removes the whole class of "AI can do things the UI can't" bugs.

```python
# portfolio/service.py  (sync; called via asyncio.to_thread or from a `def` route)
def execute_trade(conn_factory, cache, ticker, side, quantity) -> TradeResult:
    ticker = normalize_ticker(ticker)                  # upper/strip + ^[A-Z.]{1,6}$
    if quantity <= 0: raise TradeError("invalid_quantity", ...)
    price = cache.get_price(ticker)
    if price is None: raise TradeError("no_price", f"No live price for {ticker}; add it to the watchlist first")
    with closing(conn_factory()) as conn:
        conn.execute("BEGIN IMMEDIATE")                 # serializes writers; atomic read-modify-write
        cash = ...; pos = ...
        if side == "buy":
            cost = quantity * price
            if cost > cash + EPS: raise TradeError("insufficient_cash", ...)
            # new avg_cost = (old_qty*old_avg + qty*price) / (old_qty+qty); cash -= cost
        else:
            if pos is None or quantity > pos.qty + EPS: raise TradeError("insufficient_shares", ...)
            # qty -= quantity; DELETE row if qty < EPS; avg_cost unchanged; cash += proceeds
        # INSERT INTO trades ...; UPDATE users_profile ...; upsert/delete positions
        record_snapshot(conn, cache)                    # same transaction => chart point matches trade
        conn.commit()
    return TradeResult(...)
```

### Pattern 2: Reconcile tracked tickers (watchlist UNION positions)

**What:** One idempotent function computes `desired = watchlist_tickers UNION position_tickers` from the DB, diffs against `source.get_tickers()`, and calls `await source.add_ticker / remove_ticker` for the differences. Call it at startup (to build the `start()` list), after every watchlist add/remove, and after every trade that opens or fully closes a position.
**When:** Always, instead of calling `source.add_ticker` ad hoc in routes.
**Trade-offs:** One extra DB read per mutation (trivial). Eliminates drift between DB and source and the "removed from watchlist but I still own it -> price vanished -> valuation breaks" bug.

```python
async def reconcile_tracked(ctx) -> None:
    desired = await asyncio.to_thread(db_tracked_tickers, ctx.db_path)   # watchlist | positions
    current = set(ctx.source.get_tickers())
    for t in sorted(desired - current): await ctx.source.add_ticker(t)
    for t in sorted(current - desired): await ctx.source.remove_ticker(t)
```

Note: `MassiveDataSource.remove_ticker`/`SimulatorDataSource.remove_ticker` clear the cache entry, so the reference price (below) must be cleared in `PriceCache.remove` too.

### Pattern 3: Session-open / reference price lives in `PriceCache`, ships in the SSE payload

**What:** Daily change % needs a baseline the frontend cannot derive (it resets on every page reload). Put it server-side, next to the price:

- `PriceCache` gets `_reference: dict[str, float]`. In `update()`: `self._reference.setdefault(ticker, price)` (first price seen this process = session open = simulator seed price). New `set_reference(ticker, price)` overrides. `remove()` pops it. New `get_reference(ticker)`.
- `PriceUpdate` gains `reference_price: float | None = None` (defaulted, so all existing constructors/tests still work) plus property `day_change_percent` (`(price-ref)/ref*100`, 0.0 if ref falsy). `to_dict()` adds `"reference_price"` and `"day_change_percent"` (additive).
- `PriceCache.update()` builds the `PriceUpdate` with `reference_price=self._reference[ticker]`.
- `MassiveDataSource._poll_once`: after `cache.update(...)`, if `snap.prev_day` has a close, call `cache.set_reference(snap.ticker, snap.prev_day.close)` (guard with the same `try/except (AttributeError, TypeError)`). This is the "previous close when using Massive" requirement; the simulator needs no change. Verify the exact attribute name (`prev_day.close`) against the `massive` package models in the phase; the current tests mock snapshots, so add one test with a `prev_day`.

**Result:** SSE frames carry `price`, `previous_price` (tick; for flash), `direction` (tick; for flash), `reference_price`, `day_change_percent` (for the watchlist column). `GET /api/watchlist` and `GET /api/portfolio` read the same cache so first paint (before the first SSE frame) is already correct.
**Trade-off:** Simulator "daily" change resets on backend restart (baseline = seed again). Accepted per PROJECT.md key decision.

### Pattern 4: App factory + lifespan owns lifecycle ordering

```python
def create_app(settings=None, *, source=None, llm=None) -> FastAPI:
    s = settings or Settings.from_env()
    cache = PriceCache()
    source = source or create_market_data_source(cache)          # unstarted; no loop needed
    llm = llm or (MockLLMClient() if s.llm_mock else LiteLLMClient(s))
    ctx = AppContext(settings=s, cache=cache, source=source, llm=llm)

    @asynccontextmanager
    async def lifespan(app):
        init_db(s.db_path)                                        # sync, fast, idempotent (create+seed)
        await source.start(await asyncio.to_thread(db_tracked_tickers, s.db_path))
        await asyncio.to_thread(record_snapshot_now, ctx)         # first chart point exists at t=0
        snap_task = asyncio.create_task(snapshot_loop(ctx), name="snapshots")
        app.state.ctx = ctx
        try:
            yield
        finally:
            snap_task.cancel(); await suppress_cancelled(snap_task)
            await source.stop()

    app = FastAPI(lifespan=lifespan)
    app.include_router(health_router)
    app.include_router(create_stream_router(cache))               # after the router-in-factory fix
    app.include_router(portfolio_router); app.include_router(watchlist_router); app.include_router(chat_router)
    mount_frontend(app, s.static_dir)                             # LAST
    return app
```

Ordering notes: DB init before `source.start` (start list comes from DB). `Massive.start()` performs a network poll before returning - startup blocks for that call; acceptable, and it logs-and-continues on failure.
**Lazy init:** PLAN says "lazy on first request"; doing it in lifespan satisfies "no manual setup" and is deterministic. `init_db` must be idempotent (`CREATE TABLE IF NOT EXISTS`, seed only if no `users_profile` row) so an existing volume is never reseeded.

### Pattern 5: SQLite access without blocking the loop - sync `sqlite3`, short transactions, threadpool

**Recommendation:** stdlib `sqlite3`, connection-per-operation (`connect(path)`, `PRAGMA journal_mode=WAL`, `foreign_keys=ON`, `busy_timeout=5000`), service functions are plain sync functions that run one complete transaction each. Call them from `def` routes (FastAPI runs those in its threadpool) or `await asyncio.to_thread(...)` from async code.
**Why not aiosqlite:** it keeps one connection/thread but a multi-statement transaction spanning `await`s can interleave with another coroutine's statements on the same connection (two concurrent trades corrupt each other's read-modify-write). Sync functions with `BEGIN IMMEDIATE` make each trade atomic and need no asyncio lock. Extra dependency avoided.
**Do not** hold a transaction open across an `await` (e.g. across the LLM call) - ChatService does DB read -> (await LLM, no txn open) -> DB write.

### Pattern 6: Chat pipeline as five small steps (mock-friendly)

```
POST /api/chat {message}
  1 build_context   (to_thread) cash, positions+P&L (live prices), watchlist+prices, last N=20 messages
  2 llm.complete    (await)     LiteLLMClient | MockLLMClient  -> ChatLLMOutput | LLMError
  3 parse           pydantic validate; tolerant fallback: strip ``` fences; on failure -> message=raw text, no actions
  4 execute         watchlist_changes FIRST (so a same-turn "add X then buy X" can price), then trades, in order;
                    each wrapped: TradeError/ValueError -> ActionResult(status="failed", error=...)
                    then reconcile_tracked(); snapshot already taken inside execute_trade
  5 persist         (to_thread) one txn: INSERT user msg + assistant msg (actions JSON incl. failures)
  -> ChatResponse {message, actions:[ActionResult]}
```

- LLM-facing schema: make `trades` and `watchlist_changes` **required arrays that may be empty** (strict JSON-schema/structured-output modes generally require all properties present; "optional" from PLAN.md is satisfied by empty). Validate `side in {buy,sell}`, `action in {add,remove}`, `quantity > 0`. MEDIUM confidence on provider strictness; cheap insurance.
- **Do not call sync `litellm.completion` inside the async route.** The cerebras skill snippet uses sync `completion`; in an async handler that blocks the event loop for the whole LLM latency and freezes SSE for every client. Use `await litellm.acompletion(...)` (same args: `model="openrouter/openai/gpt-oss-120b"`, `response_format=ChatLLMOutput`, `reasoning_effort="low"`, `extra_body={"provider":{"order":["cerebras"]}}`) or `asyncio.to_thread(completion, ...)`.
- Failures are surfaced in the stored `actions` and response. Also render prior actions into the history sent to the LLM (e.g. append `[Executed: BUY 10 AAPL @190.50 | FAILED: SELL 5 TSLA - insufficient shares]` to assistant history content) so the model "knows" what happened on the next turn. No second LLM round-trip (keeps latency and the mock simple).
- `LLM_MOCK=true`: `MockLLMClient` returns deterministic `ChatLLMOutput` from keyword rules on the last user message (e.g. `buy <n> <TICKER>` -> one buy trade; `sell ...`; `add <TICKER>`/`remove <TICKER>`; else a fixed portfolio-summary message with no actions). Selected in `create_app()`, so **no OPENROUTER key needed and the real path is never imported/called**. Real-mode with missing key: return 503 with a clear message at request time (do not crash startup).
- HTTP mapping: LLM transport/parse-total failure -> 502; nothing persisted for that turn. Individual action failures are 200 with `status:"failed"`.

### Pattern 7: Static serving - mount last, `html=True`, guard existence

```python
def mount_frontend(app, directory):
    if directory and Path(directory).is_dir():
        app.mount("/", StaticFiles(directory=directory, html=True), name="frontend")
```

The app is a **single page** (`/`), so a true SPA fallback is unnecessary: `html=True` serves `index.html` at `/` and Next's `404.html` for misses. Do not add client-side routes (that is what would require a fallback). Because the `/` mount swallows every unmatched path, it must be registered after all API routers, and unknown `/api/*` paths will return the static 404 (acceptable). Define API routes **without trailing slashes** to avoid 307 redirects. Env `FINALLY_STATIC_DIR` (Docker sets `/app/static`) with no-op when absent so backend tests and dev runs without a built frontend still work.

## Data Flow

### Price flow (steady state)

```
Simulator tick(500ms)/Massive poll(15s) -> PriceCache.update (sets price, prev, reference-if-new)
   -> (version bump) -> SSE generator (per client, 500ms check) -> data: {ticker:{...}} frame
   -> EventSource.onmessage -> priceStore.ingest(frame) [one set() per frame]
   -> selectors re-render only the rows/panels whose ticker changed
```

### Trade flow (manual)

```
TradeBar click -> useTrade mutation -> POST /api/portfolio/trade {ticker,quantity,side}
  -> route -> PortfolioService.execute_trade (price from cache, one sqlite txn incl. snapshot)
  -> source reconcile (if position opened/closed a non-watchlist ticker)
  -> 200 {trade, portfolio}  | 400 {detail:{code,message}}
  -> frontend: setQueryData(['portfolio'], portfolio) + invalidate ['history']; inline error toast on 400
  -> Header/Heatmap/Positions recompute from positions x live prices (no polling needed)
```

### Chat flow with actions

```
ChatPanel send -> POST /api/chat -> (see Pattern 6) -> {message, actions[]}
  -> frontend appends bubbles; renders actions inline (green executed / red failed)
  -> if actions.length: invalidate ['portfolio','watchlist','history']
```

### Snapshot flow

`snapshot_loop`: `while True: await sleep(30); await to_thread(record_snapshot)`. Plus one snapshot in lifespan startup and one inside every `execute_trade`. `total_value = cash + sum(qty * price)`; if a position has no cached price (Massive pre-first-poll) fall back to `avg_cost` for that position rather than failing. `GET /api/portfolio/history` returns ascending `[{recorded_at, total_value}]` (cap to last ~2,000 rows; 30 s cadence = 2,880/day).

### Suggested JSON contracts (freeze these before frontend work starts)

```
GET  /api/portfolio  -> {cash_balance, total_value, total_unrealized_pnl,
                         positions:[{ticker, quantity, avg_cost, current_price|null, market_value,
                                     unrealized_pnl, unrealized_pnl_percent}]}
POST /api/portfolio/trade {ticker, quantity, side} -> {trade:{id,ticker,side,quantity,price,executed_at}, portfolio:{...as above}}
GET  /api/portfolio/history -> [{recorded_at, total_value}]
GET  /api/watchlist -> [{ticker, price|null, previous_price|null, reference_price|null,
                         day_change_percent|null, direction, timestamp|null}]
POST /api/watchlist {ticker} -> 201 item | 409 duplicate | 422 invalid ; DELETE /api/watchlist/{ticker} -> 204 | 404
POST /api/chat {message} -> {message, actions:[{type:"trade"|"watchlist", status:"executed"|"failed",
                            ticker, side?, quantity?, price?, action?, error?}]}
Errors: {detail:{code, message}}  (codes: insufficient_cash, insufficient_shares, no_price,
                                    invalid_quantity, invalid_ticker, ...)
```

## Frontend State Architecture

Two kinds of state, two tools - do not mix them:

| State kind | Examples | Tool | Update trigger |
|---|---|---|---|
| **Live, high-frequency, client-only** | latest ticks, per-ticker history buffers, connection status | Zustand store (`priceStore`) updated imperatively from the stream module | every SSE frame (~2 Hz) |
| **Server state, low-frequency** | portfolio, watchlist, portfolio history, chat log | TanStack Query | mutations/chat actions invalidate; `history` also `refetchInterval: 30_000` |
| **UI state** | selected ticker, chat panel open | small Zustand `ui` slice | user input |

Key rules:

1. **One EventSource** created in a module-level `connectPriceStream()` called from a single root effect (`'use client'` provider). Cleanup closes it (React StrictMode double-invokes effects in dev; without cleanup you get two connections). Never create it during render/prerender (`window` is undefined in the export build step).
2. **Connection status:** `onopen` -> green. `onerror`: if `es.readyState === EventSource.CONNECTING` the browser is auto-retrying -> yellow; if `CLOSED` (browser gives up, e.g. non-200 response) -> red **and** schedule a manual reconnect with backoff (create a new EventSource). Server `retry: 1000` handles ordinary drops. E2E "SSE resilience" test should cover both a dropped connection and a backend restart.
3. **History buffers:** `history[ticker]: {t:number, p:number}[]` capped (e.g. 1,200 points ~10 min at 2 Hz); append only if `frame.timestamp` differs from the last stored timestamp for that ticker (handles Massive's slow cadence and repeated frames). Sparklines render from this array (tiny SVG polyline is enough; no chart library instance x10). Main chart reads the selected ticker's array.
4. **Lightweight Charts constraint (MEDIUM, verify in phase):** series data must be strictly ascending, unique times (seconds). 500 ms ticks produce duplicate seconds -> either bucket to 1-second (keep last value) or use `series.update()` with the bucketed time; otherwise it throws/asserts. Same rule for the P&L chart (snapshot timestamps are naturally >= 30 s apart, but the immediate post-trade snapshot can be within the same second as a periodic one - dedupe).
5. **Selector granularity:** each watchlist row subscribes to `state.latest[ticker]` only. Header/heatmap/positions subscribe to `latest` for their held tickers, and compute `market_value = qty * price`, `total = cash + sum`. This is why portfolio data need not be polled: server positions change only on trades (invalidate), prices come from the stream.
6. **Price flash:** trigger on `direction` from the frame (not by diffing again). Apply a CSS class keyed on the frame timestamp (e.g. a `key`ed overlay or an effect that adds the class then removes it after ~500 ms with a CSS `transition: background-color 500ms`); flat ticks do nothing.
7. **Initial paint:** watchlist rows render from `GET /api/watchlist` (has price + `day_change_percent`) until the first SSE frame overwrites; sparklines start with one point and fill progressively (per spec).
8. **Static-export constraints (MEDIUM, from Next docs knowledge):** `output: 'export'`; single `app/page.tsx`; all interactive components `'use client'`; `images: { unoptimized: true }`; no Route Handlers/Server Actions/`rewrites` in the exported build. For local dev against the Python backend, make `next.config` conditional: in `development` add `rewrites()` proxying `/api/*` to `http://localhost:8000` (note: Next dev proxy may buffer/compress SSE - if it does, point `EventSource` at an env-configured absolute origin in dev and enable CORS only when `FINALLY_DEV_CORS=1`). Production same-origin needs neither.

## Anti-Patterns

### Anti-Pattern 1: Computing daily change % in the browser

**What people do:** use `change_percent` from the SSE frame, or the first tick since page load as the baseline.
**Why wrong:** the former is tick-to-tick (~0.01%); the latter resets on every reload and differs per tab.
**Instead:** server-side reference price in `PriceCache` and `day_change_percent` in the payload (Pattern 3).

### Anti-Pattern 2: Ad hoc `source.add_ticker/remove_ticker` calls in routes

**What people do:** watchlist DELETE immediately calls `source.remove_ticker`.
**Why wrong:** removes the price (and cache entry) for a ticker the user still holds; positions table shows null prices and the snapshot/total value silently drops.
**Instead:** `reconcile_tracked()` over `watchlist UNION positions`.

### Anti-Pattern 3: Separate validation for LLM trades

**What people do:** ChatService has its own cash/shares checks or manipulates DB directly.
**Why wrong:** divergence; the AI path becomes the unvalidated path.
**Instead:** both call `PortfolioService.execute_trade`.

### Anti-Pattern 4: Sync LLM call or sync sqlite on the event loop

**Why wrong:** one slow `completion()` (seconds) or a locked DB freezes all SSE clients and the market-update tasks sharing the loop.
**Instead:** `acompletion` / `to_thread`; sync services in `def` routes or `to_thread`.

### Anti-Pattern 5: Holding a DB transaction across the LLM await, or multi-worker uvicorn

**Why wrong:** write-lock held for LLM latency -> trades time out; with `--workers N` each worker has its own PriceCache, simulator, and snapshot loop (N independent price universes, duplicate snapshots).
**Instead:** three short phases in ChatService; run exactly **one** uvicorn worker, no `--reload` in the container.

### Anti-Pattern 6: Schema files under a directory that the volume shadows

See Structure Rationale (`backend/db/` vs `/app/db`).

### Anti-Pattern 7: Depending on module-level singletons for the app (router, cache, source)

Already bites in `stream.py` (module-level `router`). Everything per-app hangs off `create_app()` / `AppContext`.

## Component Boundaries (who talks to whom)

| From | To | Via | Notes |
|---|---|---|---|
| Routes | Services | direct call | Routes never touch sqlite or the source directly |
| PortfolioService | PriceCache | `get_price()` (read) | Never calls the data source for prices |
| PortfolioService | db | own transaction per call | Snapshot written in the same txn as the trade |
| WatchlistService / reconcile | MarketDataSource | `add_ticker/remove_ticker/get_tickers` | Only caller of add/remove |
| ChatService | Portfolio/Watchlist services | direct call | Failures -> `ActionResult` |
| ChatService | LLMClient | `await complete()` | Protocol; Mock vs LiteLLM chosen in `create_app` |
| MarketDataSource | PriceCache | `update()`, `set_reference()` | Only writer |
| SSE router | PriceCache | `version`, `get_all()` | Read-only |
| Frontend | Backend | same-origin `fetch` + one EventSource | No CORS in prod |

## Build Order (dependencies drive phases)

```
A. Backend foundation ──► B. Portfolio+Watchlist API ──► C. Chat ─────────┐
      │                              │                                     ├─► E. Docker/scripts ─► F. E2E
      └──(contracts frozen)──► D. Frontend (scaffold+stream can start after A; panels need B; chat panel needs C)
```

1. **A. Backend foundation** (no business logic yet): market extensions (`reference_price` in cache/model/SSE + Massive prev-close + tests; fix `stream.py` router scope), `config.py`, `app/db` (schema, connection, idempotent init+seed), `create_app()` + lifespan + `AppContext`, `/api/health`, mount SSE router, static mount stub. Exit: `uvicorn app.main:app` streams prices incl. `day_change_percent`; DB created/seeded on a fresh path; app-factory test harness (tmp DB, fast source, httpx `AsyncClient`/`TestClient` with lifespan) in `conftest.py`. *Why first:* everything else depends on the factory, DB, and the frozen SSE/response contracts.
2. **B. Portfolio + Watchlist services and REST** (+ snapshots): `valuation.py`, `execute_trade` (the highest-risk logic - unit-test heavily: avg cost, fractional, sell-all deletes row, insufficient cash/shares, no-price, float epsilon), watchlist add/remove + `reconcile_tracked`, snapshot post-trade + 30 s loop + startup snapshot, history endpoint. Exit: full REST works against the simulator with pytest coverage per PLAN section 12.
3. **C. Chat** (depends on B): schemas + parse tolerance, `MockLLMClient` first (unblocks E2E and frontend chat panel), context builder, execute+persist, then `LiteLLMClient` last with a skipped-by-default live smoke test (needs real key).
4. **D. Frontend** (start scaffold + `priceStore` + EventSource + connection dot + watchlist/sparklines as soon as A lands - needs only SSE + `/api/watchlist` contract; parallelizable with B/C): then Query hooks, TradeBar, Positions, Header, Heatmap, P&L chart (need B), ChatPanel (needs C or its contract + mock). Frontend unit tests alongside.
5. **E. Packaging**: multi-stage Dockerfile (Node 20 build -> copy `out/` to `/app/static`; Python 3.12 + uv; `ENV FINALLY_DB_PATH=/app/db/finally.db FINALLY_STATIC_DIR=/app/static`; one worker; consider `--timeout-graceful-shutdown 2`), start/stop scripts (mac + PowerShell), optional compose.
6. **F. E2E**: `test/docker-compose.test.yml` with `LLM_MOCK=true` + Playwright; scenarios per PLAN section 12. Mock keyword rules from C must match scenario inputs.

**Parallelism:** after A, B and the D scaffold are independent; freeze the JSON contracts above in `planning/` first so frontend and backend agents do not drift.

## Scaling Considerations

Single-user, single-container by design; scaling is not a goal. Only things that break first:

| Concern | Now | If it ever mattered |
|---|---|---|
| SSE fan-out | one generator per tab, each serializing all tickers every 500 ms | serialize once per cache version and share (broadcast) |
| `portfolio_snapshots` growth | 2,880 rows/day | cap history query / downsample / prune > N days |
| Multi-user | `user_id="default"` column already present | would need per-user auth + per-user trade locks; PriceCache stays shared |
| Multi-process | not supported (in-memory cache + source per process) | external pub/sub + single market-data worker |

## Integration Points

### External Services

| Service | Pattern | Gotchas |
|---------|---------|---------|
| OpenRouter -> Cerebras via LiteLLM | `acompletion(model="openrouter/openai/gpt-oss-120b", response_format=PydanticModel, reasoning_effort="low", extra_body={"provider":{"order":["cerebras"]}})`; `PydanticModel.model_validate_json(content)` | Async only inside the route; load `OPENROUTER_API_KEY` from project-root `.env` for local runs (python-dotenv, explicit path) while Docker uses `--env-file`; wrap in timeout (~30 s) -> 502; model output may still be malformed -> tolerant parse |
| Massive REST | existing poller | free tier 5 calls/min -> 15 s; new tickers have no price up to 15 s; unknown tickers never get a price (trade rejected `no_price`) |

### Other cross-cutting gotchas encountered while mapping boundaries

- **Ticker validation is ours:** simulator accepts any string and invents a price in 50-300, so a typo ("AAPLL") becomes a tradable stock. Validate `^[A-Z][A-Z0-9.\-]{0,9}$` after upper/strip in one `normalize_ticker()` used by REST, watchlist and chat. (Optionally restrict simulator mode to known `SEED_PRICES` plus pattern-valid tickers; product decision for the roadmap.)
- **Float money:** columns are REAL per spec. Use an epsilon (1e-9) for "enough cash/shares" comparisons, delete positions whose qty < epsilon after a sell, round displayed values only at the edge.
- **Simulator restart discontinuity:** prices reset to seed on restart while persisted positions/avg_cost reflect the previous random walk, so P&L can jump. Acceptable for a demo; document it so E2E tests and the demo script do not assume continuity across restarts.
- **Graceful shutdown with live SSE:** infinite generators can make `docker stop` wait out its timeout; set a short `--timeout-graceful-shutdown` and rely on `is_disconnected()`.
- **`.env` discovery:** PLAN says "reads `.env` from project root"; resolve relative to the repo root (not CWD, since `uv run` is typically launched from `backend/`), and treat Docker `--env-file` env vars as authoritative.

## Sources

- Direct code reading: `backend/app/market/{interface,cache,models,stream,simulator,massive_client,factory,__init__}.py`, `backend/pyproject.toml`, `backend/tests/market/test_models.py` (HIGH)
- `planning/PLAN.md`, `.planning/PROJECT.md`, `.planning/codebase/ARCHITECTURE.md`, `planning/MARKET_DATA_SUMMARY.md` (HIGH, project contract)
- `.claude/skills/cerebras` skill (LiteLLM + OpenRouter + Cerebras call shape) (HIGH for call shape; async adaptation is our recommendation)
- Next.js static export constraints, EventSource readyState semantics, Lightweight Charts ascending-time requirement, strict structured-output schema behavior: general ecosystem knowledge, not re-verified against live docs in this session (MEDIUM) - verify during the relevant phase's research step

---
*Architecture research for: AI-powered simulated trading workstation*
*Researched: 2026-10-27*
