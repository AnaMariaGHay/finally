# Pitfalls Research

**Domain:** AI-powered simulated trading workstation (FastAPI + SSE + SQLite + LiteLLM/OpenRouter/Cerebras structured outputs + Next.js static export + single-container Docker + Playwright E2E), built brownfield on an existing, tested market-data backend, developed on Windows.
**Researched:** 2026-10-27
**Confidence:** MEDIUM-HIGH. Findings grounded in the actual repo (`stream.py`, `cache.py`, `simulator.py`, `pyproject.toml`, git config, PLAN.md) are HIGH. OpenRouter structured-output behaviour verified against OpenRouter docs (HIGH). Next.js, Lightweight Charts, EventSource, SQLite, Docker and Playwright items rest on established behaviour and training knowledge (MEDIUM). Items marked **[VERIFY]** need a short spike before relying on them.

**Phase labels used below** (map onto the real roadmap as needed):
- **P1** Backend platform: app entrypoint, DB layer, watchlist, portfolio/trade service, snapshots
- **P2** LLM chat
- **P3** Frontend
- **P4** Docker and start/stop scripts
- **P5** E2E and hardening

---

## Critical Pitfalls

### Pitfall 1: `StaticFiles` mounted at `/` shadows `/api` (or crashes when `static/` is absent)

**What goes wrong:**
`app.mount("/", StaticFiles(directory="static", html=True))` is a catch-all. If it is registered before the routers, or before the SSE route, every `/api/*` request hits the file server and returns a 404 HTML page. Separately, `StaticFiles` raises `RuntimeError: Directory 'static' does not exist` at construction, so the backend cannot start in local dev or in pytest until the frontend has been built. Unknown `/api/foo` paths also return the HTML 404 instead of JSON.

**Why it happens:**
Mount order is route order in Starlette. Agents write the static mount first because it "is the frontend" and add routers afterwards. The directory is relative to the process CWD, so it works from one folder and not from another.

**How to avoid:**
- Build the app in a factory (`create_app()`). Include all routers first (`/api/*`, including the SSE router). Mount static last.
- Mount only `if STATIC_DIR.is_dir()`. Resolve the path from `Path(__file__)` or a `STATIC_DIR` env var, never from CWD.
- Register an `/api/{path:path}` JSON 404 catch-all before the static mount.
- Add a pytest that asserts `GET /api/health` returns JSON with the static mount present.

**Warning signs:**
`/api/health` returns `text/html`. Backend tests fail with "Directory 'static' does not exist". The page loads but every fetch returns HTML.

**Phase to address:** P1 (app entrypoint), re-verified in P4.

---

### Pitfall 2: Existing SSE generator swallows cancellation and blocks graceful shutdown

**What goes wrong:**
`backend/app/market/stream.py` loops forever and catches `asyncio.CancelledError` without re-raising. Two consequences:
1. Uvicorn's shutdown waits for open SSE connections. `docker stop` hangs the full 10s and ends in SIGKILL, so lifespan shutdown code (stop the simulator, flush the DB) may never run. This also slows every test and Playwright teardown.
2. Swallowed cancellation can leave tasks half-cancelled.

Related defect in the same file: `router = APIRouter(...)` is module-level, and `create_stream_router()` attaches `@router.get("/prices")` to it on every call. Building the app twice (tests, app factory, reload) registers duplicate routes on the same shared router object.

**Why it happens:**
The module was built and tested standalone with no lifespan and no app factory. `request.is_disconnected()` is only polled between sleeps, and an idle `await asyncio.sleep(0.5)` gives uvicorn nothing to interrupt.

**How to avoid:**
- Re-raise `CancelledError` (or drop the `except` and use `finally` for logging).
- Create the `APIRouter` inside the factory function.
- Add a shutdown flag (`app.state.shutting_down` as an `asyncio.Event`). The generator checks it each loop and exits. Set it first thing in lifespan teardown. Also run uvicorn with `--timeout-graceful-shutdown 3` or similar.
- Emit a `: ping\n\n` comment heartbeat every ~15s. Currently the stream sends nothing when the cache version is unchanged, which is the normal case with Massive's 15s polling. Idle proxies drop it, and clients cannot distinguish "quiet" from "dead".
- Write the missing SSE tests (see Pitfall 17).

**Warning signs:**
`docker stop` takes about 10s. "Application shutdown complete" never appears in logs. `Active connections` grows after tab closes. Two identical routes appear in `app.routes`.

**Phase to address:** P1.

---

### Pitfall 3: Wrong "daily change %" source: the SSE `change_percent` is tick-to-tick

**What goes wrong:**
`PriceUpdate.change_percent` and `direction` compare against the previous tick (about 500ms ago). If the watchlist's "daily change %" column reads it, it flickers around +/-0.01% and means nothing. The PROJECT.md decision is session-open baseline (simulator) or previous close (Massive). If the frontend computes a baseline from the first SSE message it sees, the number resets on every page refresh.

**Why it happens:**
The field is named `change_percent` and is already in the payload, so it is the path of least resistance. The baseline needs server-side state that the existing cache does not hold.

**How to avoid:**
- Store `session_open_price` per ticker in the backend. It is set when the ticker is first priced, or from Massive's previous close. Expose it as `open_price` / `day_change_percent` on the SSE payload and on `GET /api/watchlist`.
- Keep the tick-based `direction` for flash only.
- Name the fields distinctly (`tick_change` vs `day_change_percent`) in the API contract (document it in `planning/`).

**Warning signs:**
Daily change % jumps sign constantly. It differs after a page reload. A freshly added ticker shows 0.00% forever.

**Phase to address:** P1 (contract), P3 (consume).

---

### Pitfall 4: Flash animation keyed on `direction` re-fires on unchanged tickers

**What goes wrong:**
The stream sends a full snapshot of all tickers whenever any price bumps the cache version, and each ticker's `direction` is its last tick's direction. A ticker that did not move this round still arrives with `direction: "up"`. A frontend that flashes on `direction` flashes continuously, or flashes the wrong ticker. Re-adding the CSS class while it is already present does not restart the transition, so flashes get skipped or stick.

**Why it happens:**
`direction` looks like the obvious signal. The snapshot-not-delta payload shape is easy to forget.

**How to avoid:**
- Flash only when `price !== lastRenderedPrice` (or `timestamp` advanced) for that ticker. Store the previous price in the client store, not React state.
- Restart the animation reliably: toggle a data attribute or key, or use the Web Animations API (`el.animate`), rather than add/remove class in the same frame.
- Unit-test with the same price sent twice (must not flash) and an up then down sequence.

**Warning signs:**
Whole table flashes green every tick. Flash looks "stuck". Flash on page load for every row.

**Phase to address:** P3.

---

### Pitfall 5: EventSource status dot lies (reconnecting vs closed vs silently dead)

**What goes wrong:**
- `onerror` fires for both a transient drop (browser auto-retries, `readyState === CONNECTING`) and a fatal failure (non-200 or wrong content-type, `readyState === CLOSED`). In the CLOSED case the browser never retries. A naive "red on error" implementation is wrong half the time, and one that assumes auto-retry leaves a permanently dead page.
- A proxied or stalled connection can stay "open" with no data. The dot stays green while prices are frozen.
- Prices from before the drop remain displayed as if live.
- React StrictMode (dev) mounts effects twice and opens two EventSources. Components that each open their own EventSource multiply connections, and HTTP/1.1 allows about 6 per origin, so multiple tabs or components starve normal fetches.

**How to avoid:**
- One module-level singleton connection feeding an external store. Components never create an EventSource. Close on teardown.
- State machine: `onopen` -> green; `onerror` with `readyState===CONNECTING` -> yellow; `readyState===CLOSED` -> red and schedule a manual recreate with backoff.
- Watchdog timer: no message and no heartbeat within about 5s -> yellow ("stale"). This is why the server heartbeat in Pitfall 2 matters.
- On reconnect, expect sparkline gaps. Do not interpolate fake points.

**Warning signs:**
Dot is green but prices are frozen. After backend restart the page never recovers. Network tab shows 2+ pending `/api/stream/prices`.

**Phase to address:** P3 (client), P1 (heartbeat), P5 (reconnect E2E).

---

### Pitfall 6: 500ms ticks cause a re-render storm across the whole dashboard

**What goes wrong:**
Putting prices in React Context or in top-level `useState` re-renders the watchlist, positions table, header, treemap, charts and chat panel twice per second. Chart components that call `setData` on each tick, or Recharts with animation enabled, burn CPU. The tab becomes sluggish within minutes, and Playwright actions flake because elements re-render mid-click.

**Why it happens:**
The natural React shape is "SSE hook returns prices, pass down as props".

**How to avoid:**
- Prices live in an external store (`useSyncExternalStore` or zustand) with per-ticker selectors. A row subscribes only to its own ticker.
- Header total value subscribes to a derived selector, not the whole map. Throttle derived/aggregate views (treemap, P&L) to about 1Hz or to trade events.
- `React.memo` rows. Disable chart animations (`isAnimationActive={false}` in Recharts).
- Apply the flash by direct DOM class/attribute manipulation or a tiny per-cell component, not by re-rendering the table.
- Check with React DevTools Profiler while the stream is running before declaring P3 done.

**Warning signs:**
CPU above 30% idle with the page open. Profiler shows the root re-rendering every tick. Typing in the chat box lags.

**Phase to address:** P3 (store architecture must be decided before building components).

---

### Pitfall 7: Chart library leaks, ordering errors and SSR crashes

**What goes wrong:**
- Lightweight Charts: `chart.remove()` is not called on unmount. StrictMode double-mounts leave orphan canvases and ResizeObservers, and memory grows on every ticker selection.
- Lightweight Charts requires strictly ascending integer-second timestamps. 500ms ticks produce duplicate seconds, and `update()` with an older time throws ("Cannot update oldest data"). `setData` with unsorted or duplicate times throws. Portfolio snapshots taken by a trade and by the 30s timer within the same second hit the same bug.
- API changed between major versions (v4 `addLineSeries()` vs v5 `addSeries(LineSeries)`). Agent training data mixes both.
- Unbounded sparkline/price arrays grow forever (about 7,200 points/hour/ticker).
- Ten Lightweight Chart instances for sparklines is heavy. Recharts is SVG, not canvas, despite the PLAN.md note.
- Importing a chart lib that touches `window` breaks `next build` prerender ("window is not defined").
- A treemap with zero positions or zero-value nodes renders nothing or NaN.

**How to avoid:**
- Pin one chart library version and write down the API in the plan.
- Wrap each chart in a client component that creates the chart in `useEffect` and returns `chart.remove()`. Use `dynamic(..., { ssr: false })` from a Client Component for anything touching `window`.
- Convert timestamps to integer UTC seconds. Keep latest value per second (replace, don't append). Use `series.update()` after the initial `setData`.
- Cap series per ticker at a fixed ring buffer (about 300-600 points).
- Use a lightweight SVG polyline for the watchlist sparklines. Reserve the full chart for the main chart and P&L chart.
- Empty states for treemap and P&L chart. Seed one portfolio snapshot at backend startup so the P&L chart is never empty on first load.

**Warning signs:**
Heap grows when clicking between tickers. Console errors about time ordering. `next build` fails with `window is not defined`. Duplicate canvases in the DOM.

**Phase to address:** P3.

---

### Pitfall 8: Float money math, average cost and "sell everything"

**What goes wrong:**
- `qty * price` compared to `cash` in float: a "buy max" yields `cost` of cash + 1e-13 and is rejected, or leaves cash at -1e-12 and displays "-0.00".
- Selling an entire fractional position leaves a dust position (0.30000000000000004 - 0.1 - 0.2 = 5e-17) that renders as a phantom row.
- Average cost wrongly recomputed on sells. Correct rule: average cost changes only on buys, `new_avg = (q*avg + n*p) / (q + n)`. Sells reduce quantity and leave avg_cost unchanged. When quantity reaches 0, the row is deleted.
- Price comes from the cache already rounded to 2 dp. Recomputing P&L from unrounded values elsewhere makes backend and frontend disagree by cents.
- NaN/Infinity/negative/zero quantity from the LLM or API. `float("nan") > 0` is False but `nan < cash` is also False, so checks pass or fail inconsistently.

**How to avoid:**
- One `execute_trade` service used by both REST and chat. Compute in `Decimal`: `cost = (Decimal(str(qty)) * Decimal(str(price))).quantize(Decimal("0.01"))`. Use that single `cost` for both the check and the update. SQLite columns stay REAL per spec; round on write (cash to 2 dp, quantity to 4-6 dp).
- Quantity bounds in Pydantic: `gt=0`, finite (`allow_inf_nan=False`), sane max, max decimals.
- If `remaining < 1e-9` after a sell, delete the position row.
- Reject trades when the ticker has no cached price (just-added ticker not priced yet).
- Put the P&L formula in one place and unit-test known cases: buy, buy higher, sell partial, sell all, sell at a loss, buy with exact cash, fractional.
- Client-side live valuation (Pitfall 9) must reuse the same formula. Add a parity test.

**Warning signs:**
"-0.00" or "-$0.00" in cash. Positions table rows with quantity 0.00. P&L off by a cent versus backend.

**Phase to address:** P1.

---

### Pitfall 9: Trade is not atomic; concurrent trades corrupt cash

**What goes wrong:**
A trade is read cash -> check -> update position -> update cash -> insert trade -> snapshot. With a manual trade and an LLM batch running in the same event loop (or a double-clicked Buy), two requests read the same cash and both succeed. Separately, partial failure midway leaves positions updated but cash not.

**How to avoid:**
- Do the whole trade in one SQLite transaction started with `BEGIN IMMEDIATE`. Python's sqlite3 implicit transaction handling is surprising, so set `isolation_level=None` and issue explicit `BEGIN IMMEDIATE`/`COMMIT`/`ROLLBACK`.
- Also serialize trade execution with one `asyncio.Lock` in the portfolio service, so LLM multi-trade batches are sequential.
- Take the portfolio snapshot after commit, using the cached price.
- Disable Buy/Sell/Send buttons while a request is in flight on the client.
- Add a `CHECK (cash_balance >= 0)` and `CHECK (quantity > 0)` constraint as a backstop.
- Test with `asyncio.gather` of two buys that together exceed cash.

**Warning signs:**
Negative cash after rapid clicks. `trades` rows with no matching position change.

**Phase to address:** P1.

---

### Pitfall 10: SQLite from async code: blocking, thread errors, WAL on volumes, init races

**What goes wrong:**
- Sync `sqlite3` calls inside `async def` handlers block the loop. Commit `fsync` on a Docker Desktop volume can take tens of ms, which stalls SSE ticks.
- A connection created in one thread and used via `asyncio.to_thread` raises "SQLite objects created in a thread can only be used in that same thread", or corrupts under concurrent use if `check_same_thread=False` without a lock.
- `database is locked` from a second connection (snapshot background task vs request handlers) with default 0 timeout.
- WAL mode needs shared memory and fails or misbehaves on some bind mounts (notably Windows-host bind mounts through Docker Desktop). Named volumes are fine.
- Lazy "first request" initialization races: two concurrent first requests both create and seed (duplicate default tickers or a UNIQUE violation).
- Schema changes during development: the volume persists the old DB, "tables exist so skip init" never migrates, and the app fails on missing columns.
- Windows tests: removing a temp `.db` while a connection is open raises PermissionError.

**How to avoid:**
- Use `aiosqlite` (one worker thread per connection, serialized), or one dedicated connection plus a lock via `to_thread`. Pick one and wrap it in a small repository class. Do not mix.
- Initialize in FastAPI `lifespan` startup, not on first request. Use `CREATE TABLE IF NOT EXISTS` and `INSERT OR IGNORE`.
- `PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL; PRAGMA busy_timeout=5000; PRAGMA foreign_keys=ON`. Fall back to default journal mode if the WAL pragma does not return `wal`.
- Use `PRAGMA user_version` for schema version. Mismatch in dev -> clear error telling the developer to remove the volume.
- Tests use `tmp_path` DBs and close connections in fixtures.
- Run exactly one uvicorn worker (in-memory cache and simulator are per-process).

**Warning signs:**
SSE cadence hiccups during writes. Intermittent `database is locked`. Duplicate watchlist rows or crash on a fresh volume.

**Phase to address:** P1.

---

### Pitfall 11: Hidden coupling between watchlist, market data source and positions

**What goes wrong:**
- Removing a ticker from the watchlist stops the source tracking it and drops it from the `PriceCache`. If a position exists, the portfolio valuation has no price: total value silently drops, the heatmap shows NaN, snapshots record a wrong total, and trades on it are rejected.
- A newly added ticker has no cached price until the next tick (up to 15s with Massive). `GET /api/watchlist` returns `price: null`, and the frontend does `.toFixed` on null.
- The simulator's `add_ticker` accepts any string and assigns `random.uniform(50, 300)` as the seed price. A hallucinated LLM ticker or a typo becomes a fully tradeable fake stock. Massive returns no data for unknown tickers, so they stay priceless forever.
- Ticker input is currently unvalidated (also in `CONCERNS.md`).
- Source and DB drift: DB says tracked, source does not, after restart.

**How to avoid:**
- Tracked set = `watchlist UNION tickers with open positions`. Only stop tracking when neither applies. Recompute on every watchlist/trade change and at startup.
- Central `validate_ticker()` (uppercase, strip, `^[A-Z0-9.\-]{1,10}$`) at API boundary and LLM-output boundary. Return 422/400 with a clear message.
- For simulator: accept format-valid tickers (needed for "PYPL"-style adds) but document that unknown ones get synthetic prices. For Massive: add only after the first poll returns a price, otherwise reject.
- Nullable prices in API types and UI ("--" placeholder).
- At startup, seed the source from the DB (`start(watchlist + position tickers)`).

**Warning signs:**
Total portfolio value drops when removing a watchlist ticker. `null` crash in UI after adding a ticker. LLM "adds" nonsense symbols successfully.

**Phase to address:** P1.

---

### Pitfall 12: LiteLLM sync call blocks the event loop (the skill snippet is sync)

**What goes wrong:**
The cerebras-inference skill uses `from litellm import completion`, which is synchronous. Copied into `async def chat()`, every chat request freezes the entire server for the LLM latency (1-5s), freezing SSE for all clients and delaying trades. Separately, no timeout means a hung OpenRouter call hangs the request indefinitely.

**How to avoid:**
- Use `litellm.acompletion` (same kwargs) with `timeout=` (e.g. 30s) and `num_retries=1`. Keep the sync form only for scripts.
- Chat handler catches `Exception` from the LLM layer and returns a normal chat response with an error message (HTTP 200 or a structured 502), never a raw 500 stack trace.
- Lazy-import `litellm` inside the non-mock path. It is a heavy import (seconds), and mock mode and unit tests should not pay for it or need a key.
- Set `LITELLM_LOCAL_MODEL_COST_MAP=True` in the environment so LiteLLM does not fetch its cost map from GitHub on import (can stall in sandboxed/offline containers) **[VERIFY]**.
- Add a "does the event loop stay responsive during chat" test (SSE tick arrives while a mocked slow LLM call is pending).

**Warning signs:**
Prices freeze for seconds when chat is sent. Container start is slow or hangs offline.

**Phase to address:** P2.

---

### Pitfall 13: Structured output quirks via OpenRouter/Cerebras gpt-oss-120b

**What goes wrong:**
- OpenRouter documents that structured output support varies by provider endpoint, not just model, and strict compliance "is not guaranteed on every endpoint" (verified in OpenRouter docs). The skill's `provider: {"order": ["cerebras"]}` still allows fallback to other providers by default. Fallback can silently route to a provider with different latency, different JSON behaviour, or no `json_schema` support (a 400 "no endpoints found").
- gpt-oss is a reasoning model. `message.content` can be `None` or truncated (`finish_reason == "length"`) if reasoning consumes the token budget. Leave `reasoning_effort="low"` per the skill and set an explicit generous `max_tokens`.
- Strict JSON schema modes generally require all properties listed in `required` and `additionalProperties: false`. A Pydantic model with `Optional[...] = None` / default fields produces a schema that is not all-required, which some strict endpoints reject or mishandle. Unsupported schema keywords (pattern, min/max constraints, formats, deep recursion) may be rejected or ignored by Cerebras **[VERIFY]**.
- Model emits markdown fences, a preamble, or the wrong enum case ("BUY", "Buy"), or `quantity` as a string.

**How to avoid:**
- Keep the schema flat and boring: `message: str`, `trades: list[Trade]`, `watchlist_changes: list[Change]` (always present, empty list when none), `Trade{ticker: str, side: Literal["buy","sell"], quantity: float}`. Put range checks in Python after parsing, not in the JSON Schema.
- Routing: `extra_body={"provider": {"order": ["cerebras"], "allow_fallbacks": False, "require_parameters": True}}` so a mismatch fails loudly instead of degrading silently **[VERIFY against current OpenRouter docs and the skill]**.
- Layered parse, never crash: (1) `Model.model_validate_json(content)`; (2) on failure strip fences / extract first `{...}` and retry; (3) on failure, optionally one retry with `json_object` mode plus schema in the prompt; (4) final fallback is `message` = raw text, no trades. Always persist and return something.
- Normalize after parse: lowercase `side`, uppercase `ticker`, coerce numeric strings, drop invalid items individually (one bad trade does not discard a valid one).
- Pin the `litellm` version in `uv.lock`; add one opt-in live smoke test (`@pytest.mark.live`, skipped without a key) that exercises the real schema end to end before declaring P2 done.

**Warning signs:**
`content is None`. `ValidationError` on `trades.0.side`. 400/404 "No endpoints found" from OpenRouter. Latency suddenly higher (fallback provider).

**Phase to address:** P2. Likely needs a short spike with the real key before planning (see Research Flags).

---

### Pitfall 14: Chat says "Bought 10 AAPL" but the trade failed (or ran twice)

**What goes wrong:**
In the single-call design the LLM writes its message before the server executes anything. If the trade then fails validation (insufficient cash), the displayed message still claims success. Further, persisted assistant history containing "I bought 10 AAPL" is fed back to the model next turn, and it may re-emit the same trade (duplicate execution). The user's "yes" after a suggestion is ambiguous, and a hypothetical question ("what if I bought 100 TSLA?") can be interpreted as an instruction. Chained trades in one response can collectively exceed cash even though each looked fine individually.

**How to avoid:**
- The server is the source of truth for outcomes. Return `actions` as structured results (`{type, ticker, side, quantity, price, status: "executed"|"failed", error}`) and have the UI render the inline confirmations from those, not from the model text. Append a deterministic one-line failure summary to the displayed message when any action failed.
- Store `actions` separately (the schema already has the column). When building history for the model, include only prior message text plus a short neutral tag like `[executed: BUY 10 AAPL @ 190.12]`, and instruct "do not repeat trades from earlier turns; trade only on this message's explicit request or clear agreement".
- Execute trades sequentially, sells before buys is a reasonable ordering. Cap trades per response (e.g. 5) and ignore the rest with a note.
- Prompt guidance: only include trades when the user explicitly asks or agrees; questions and hypotheticals produce analysis only.
- Cap history sent to the model (last ~20 messages) and keep the portfolio context compact.
- Frontend disables Send while loading, to prevent double-submit.

**Warning signs:**
Chat text and trade confirmation chips disagree. Same trade appears twice in `trades` after a "thanks". Cash goes unexpectedly low after a casual question.

**Phase to address:** P2.

---

### Pitfall 15: Docker volume masks code; `backend/db/` vs top-level `db/` confusion

**What goes wrong:**
PLAN.md puts schema/seed code in `backend/db/` and mounts the runtime volume at `/app/db`. If the Dockerfile copies `backend/` to `/app`, then `backend/db/` (schema code) lands at `/app/db`, exactly the mount point. A named volume is populated from image content on first use, so stale schema code gets frozen into the volume and persists across image rebuilds. A bind mount shadows it, so the schema files vanish and init fails. Locally, running `uvicorn` from `backend/` with a CWD-relative `db/finally.db` writes the database into `backend/db/`, mixing runtime data with source.

**How to avoid:**
- Do not keep the schema as loose files in the mounted directory. Embed it as a Python module/string (e.g. `app/db/schema.py`), or copy the backend to `/app/backend` and use `/app/db` for data only.
- One `DB_PATH` env var, defaulting to an absolute path derived from the repo root locally and `/app/db/finally.db` in the container. Never CWD-relative.
- `.gitignore` `db/*.db*` (including `-wal`, `-shm`) while keeping `db/.gitkeep`.
- Document "reset" as `docker volume rm finally-data`.

**Warning signs:**
Schema edits don't take effect after rebuild. `finally.db` appears under `backend/`. Container starts fine on a fresh volume but fails with a bind mount.

**Phase to address:** P1 (path decision), P4 (Dockerfile).

---

### Pitfall 16: Windows line endings break shell scripts and containers

**What goes wrong:**
This machine has `core.autocrlf=true` and the repo has no `.gitattributes`. `start_mac.sh` is checked out with CRLF, and running it (or any shell script copied into a Linux image) fails with `/bin/bash^M: bad interpreter` or `$'\r': command not found`. `.env` with CRLF can leave a trailing `\r` on values, so `LLM_MOCK=true\r` is not `"true"` and mock mode silently turns off (or the API key is rejected). Git on Windows also drops the executable bit. Git Bash (MSYS) rewrites `-v /app/db` style args into Windows paths when invoking `docker`. PowerShell scripts blocked by execution policy.

**How to avoid:**
- Add `.gitattributes` immediately: `*.sh text eol=lf`, `Dockerfile text eol=lf`, `*.ps1 text eol=crlf` (or lf; PowerShell tolerates both), `*.py text eol=lf`. Then `git add --renormalize .`.
- `git update-index --chmod=+x scripts/*.sh` and document `bash scripts/start_mac.sh` as the fallback.
- Avoid shell entrypoint scripts in the image. Use `CMD ["uvicorn", ...]` directly.
- Parse env values defensively in code: `.strip().lower() == "true"`, `.strip()` on keys.
- Note that `docker run --env-file` does not strip quotes (`KEY="abc"` keeps the quotes). Document unquoted `.env` and ship `.env.example` that way.
- Tell Windows users: `powershell -ExecutionPolicy Bypass -File scripts/start_windows.ps1`. Set `MSYS_NO_PATHCONV=1` if testing scripts from Git Bash.
- Make scripts idempotent as the spec requires: `docker rm -f finally 2>/dev/null`, build only if the image is absent or `--build`, check port 8000 is free.

**Warning signs:**
`bad interpreter` error. Mock mode "on" in `.env` but real LLM calls happen. 401 from OpenRouter with a seemingly correct key.

**Phase to address:** P4 (add `.gitattributes` in the first commit of P1 so no file is born with CRLF).

---

### Pitfall 17: Backend test traps for SSE and lifespan

**What goes wrong:**
`httpx.AsyncClient(transport=ASGITransport(app))` buffers the entire response body, so an infinite SSE endpoint hangs the test forever. It also does not run lifespan, so `app.state` (DB, price cache, source) is missing and routes 500. `TestClient` only runs lifespan inside `with TestClient(app) as client:`. Fixture ordering with `pytest-asyncio` function-scoped loops can leave background tasks running between tests.

**How to avoid:**
- Test `_generate_events` directly: feed a fake `Request` whose `is_disconnected()` flips, read N chunks with `anext()`, assert `retry:` first, assert payload shape, version-gated emission, and clean exit.
- For API tests, use `with TestClient(create_app(test_settings)) as client` so lifespan runs, with `LLM_MOCK=true`, a temp DB path, and a deterministic fake market source.
- One real-server smoke test (uvicorn in a thread on an ephemeral port, `httpx.stream`) for headers (`text/event-stream`, no `Content-Encoding`).
- Do not add `GZipMiddleware` (buffering SSE). If compression is ever added, exclude `text/event-stream`.

**Warning signs:**
Test suite hangs. Routes 500 with `AttributeError: state has no attribute ...`.

**Phase to address:** P1.

---

## Moderate Pitfalls

### Pitfall 18: Next.js static export limits discovered late

**What goes wrong:**
- `output: 'export'` supports no API routes / route handlers that depend on requests, no middleware, no `rewrites/redirects/headers`, no server actions, no ISR, no default `next/image` optimizer (build error unless `images: { unoptimized: true }`).
- Dynamic routes (`/ticker/[symbol]`) require `generateStaticParams`, impossible for user-added tickers. Store the selected ticker in state or a query param on the single page.
- Hard-coded `http://localhost:8000` or a build-time `NEXT_PUBLIC_API_URL` bakes the wrong origin into the bundle. All calls must be relative `/api/...`.
- Dev experience: `next dev` on :3000 can't reach the API on :8000 without a rewrite, but rewrites don't apply to export builds. Enable the dev rewrite conditionally on `NODE_ENV === 'development'`, or run dev by building and serving via FastAPI.
- Tailwind v4 vs v3 mismatch: v4 is CSS-first (`@import "tailwindcss"`, `@theme`, `@tailwindcss/postcss`); agents often emit a v3 `tailwind.config.js`, producing an app with no styles and no build error.
- Hydration mismatches from `Date.now()`, locale formatting or `window` access in components prerendered at build time.
- Rendering LLM text with `dangerouslySetInnerHTML` (XSS). Render as text.

**How to avoid:**
Set `output: 'export'`, `images.unoptimized: true`, no `app/api` directory, a single page with client state, `export const dynamic = 'error'` in the root layout to fail the build on dynamic features. Check the installed Tailwind major version and use the matching setup. Run `next build` in CI of P3 and serve `out/` from FastAPI before building any further UI.

**Warning signs:**
`next build` errors mentioning "export", unstyled UI, 404 on refresh of a sub-path.

**Phase to address:** P3 (scaffold step), P4 (copy `out/` to `static/`).

---

### Pitfall 19: Portfolio numbers drift between REST and live SSE; chat/watchlist actions don't refresh UI

**What goes wrong:**
Header "total value updating live" needs positions x live price, but positions come from `GET /api/portfolio` (not SSE). Either the header is stale until the next poll, or the client calculates it differently from the backend. After an LLM trade or watchlist change, the UI shows old positions/watchlist because only the manual trade bar triggers a refetch.

**How to avoid:**
Client computes live value as `cash + sum(qty * livePrice)` from the last fetched portfolio, using the same formula as the backend (parity test). Refetch portfolio, watchlist and history after any manual trade and after any chat response with non-empty `actions`. Define a single `refreshAll()` and call it from both paths.

**Warning signs:**
Chat-bought position doesn't appear until reload. Header total differs from sum in the positions table.

**Phase to address:** P3.

---

### Pitfall 20: Snapshots and P&L chart edge cases

**What goes wrong:**
Fresh start has zero snapshots (empty chart). Two snapshots in the same second (trade + timer) break chart ordering. Background task writes through a different DB connection and hits locks. Snapshot loop dies silently on one exception and never restarts. Timestamps stored without timezone are parsed as local time by the browser. Unbounded growth (30s x 24h = 2,880 rows/day, fine, but the endpoint should cap or downsample).

**How to avoid:**
Seed a snapshot at startup. Wrap the loop body in try/except with logging. Store ISO-8601 UTC with `Z`. Dedupe by second server-side or client-side. Use the shared DB wrapper. Limit `/api/portfolio/history` (e.g. last 1,000 points).

**Phase to address:** P1 (backend), P3 (chart).

---

### Pitfall 21: Docker build pitfalls specific to this repo

**What goes wrong:**
- No `.dockerignore`: `COPY backend/` brings the Windows `.venv` (with `Scripts/`, Windows interpreter paths), `__pycache__`, and the host `node_modules`/`.next`. `uv sync` then misbehaves. A stray `.env` gets baked into the image layers (secret leak).
- `pyproject.toml` has `readme = "README.md"` and hatchling `packages = ["app"]`. `uv sync` installs the project itself and fails if `README.md` or `app/` aren't copied yet. Copying only `pyproject.toml` + `uv.lock` first for layer caching therefore breaks, unless using `uv sync --frozen --no-install-project`, then copying source and syncing again.
- uvicorn defaults to `127.0.0.1`, so the published port is unreachable. Must be `--host 0.0.0.0`.
- `python:3.12-slim` has no `curl`/`wget`, so a `curl` HEALTHCHECK fails. Use `python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/api/health')"`.
- Multiple workers (`--workers`) create multiple simulators with divergent prices. Always one.
- `node:20-slim` must satisfy the installed Next.js minimum Node version. Check the `engines` field after scaffold.
- Frontend `npm ci` needs a committed `package-lock.json`. Build order matters: static export must exist before the final `COPY --from=frontend /app/out ./static`.
- Running as root with a volume: acceptable here, but file ownership on bind mounts can bite on Linux.

**How to avoid:**
Create `.dockerignore` (`.venv`, `**/__pycache__`, `node_modules`, `.next`, `out`, `.env`, `db/*.db*`, `.git`, `.planning`). Use `uv sync --frozen --no-dev` with `UV_LINK_MODE=copy`. Add a Docker health check and `docker run` smoke test (curl `/api/health` and `/`, assert `/api/stream/prices` first chunk) as the P4 exit criterion.

**Warning signs:**
"python: not found" or exec format errors inside the container. Port published but connection reset. Image contains `.env`.

**Phase to address:** P4.

---

### Pitfall 22: Playwright flakiness from live streaming data and shared state

**What goes wrong:**
- Prices are random (GBM, `np.random` unseeded). Exact-value assertions fail. "Price is streaming" tests that compare two snapshots can pass or fail by chance (a ticker may not tick in the window).
- Tests share one container and one SQLite DB. Order-dependent state ("fresh start has $10,000") fails after a buy test ran first.
- Acting before the first SSE message arrives: clicking Buy before prices exist yields "no price" rejection.
- Flash animations and 2Hz DOM updates detach elements mid-click.
- Asserting on animation timing (500ms fade) is inherently flaky.
- Mock LLM responses that depend on market state (e.g. "sell all") are non-deterministic. A mock buy that exceeds cash breaks later tests.
- SSE reconnect test: `context.setOffline(true)` may not terminate an already-open streaming connection **[VERIFY]**, so the "disconnect" never happens and the test passes vacuously or hangs.
- Playwright image version must match the `@playwright/test` version, otherwise browser binary mismatch errors.
- Compose startup race: Playwright starts before the app is healthy.

**How to avoid:**
- Fresh state per run: tests run with `workers: 1` against a container started with a fresh/tmp volume, plus an env-gated `POST /api/test/reset` (enabled only when `E2E=true` / `LLM_MOCK=true`) called in `beforeEach`. Never expose it otherwise.
- Wait for readiness first: connection dot green and at least one price rendered (`data-testid`s everywhere; `expect(...).toBeVisible()` auto-waits).
- Assert relationships, not values: after buying 1 AAPL, cash decreases by roughly quantity x displayed price (tolerance), position row exists, trade chip appears. Use `expect.poll`.
- Mock LLM keyed on message text: e.g. "buy" -> fixed `{ticker: "AAPL", side: "buy", quantity: 1}`, "watch PYPL" -> add PYPL, anything else -> fixed reply. Small quantities so state never runs out. Document the mapping in `planning/`.
- Optional deterministic simulator seed via env (`SIM_SEED`) for E2E only.
- Test the flash by asserting a class/attribute is applied, not timing.
- SSE reconnect: use `page.route('**/api/stream/prices', r => r.abort())` plus a test-only endpoint that closes active streams, or stop/start the app container; verify yellow/red then green. Spike this.
- Compose: `depends_on: condition: service_healthy`, `baseURL=http://app:8000`, pin Playwright image tag to the npm package version.

**Warning signs:**
Tests pass alone, fail in the suite. Failures only on CI. Retries needed to go green.

**Phase to address:** P5. The mock-response contract and reset endpoint must be designed in P2.

---

## Minor Pitfalls

### Pitfall 23: Logging, lifespan and config details

- `basicConfig` is not configured: add it in the app factory, level from `LOG_LEVEL`.
- Use `lifespan`, not deprecated `@app.on_event`, even though `CONCERNS.md` suggests `on_event`.
- `.env` loading: add `python-dotenv` (or pydantic-settings) explicitly. It is not in `pyproject.toml` today, and locally nothing loads the root `.env` otherwise.
- New deps to add: `litellm`, `pydantic` (already via FastAPI), `aiosqlite`, `python-dotenv`. Check `uvicorn[standard]` installs cleanly on Windows (uvloop is skipped automatically).

### Pitfall 24: Existing simulator and Massive carry-over risks (from CONCERNS.md)

- Cholesky can fail when tickers are added dynamically. Wrap and fall back to uncorrelated (already specified in CONCERNS.md). More relevant now that users and the LLM add arbitrary tickers.
- Massive `to_thread` call has no timeout. Wrap with `asyncio.timeout`. Only matters with a key set, but it would freeze the price feed.
- Unbounded GBM growth is a non-issue for a session-length demo. Do not spend time on it.
- Rate limiting (`slowapi`) is out of scope for a single-user localhost app. Skip, but keep a basic max length on chat message input.

### Pitfall 25: UI formatting traps

- `-0.00` display; format with a helper that normalizes negative zero.
- Treemap color scale: clamp P&L % (e.g. +/-5%) so one outlier doesn't flatten the palette. Do not rely on color alone (add +/- sign).
- Chat panel autoscroll should not fight the user reading history. Enter key double-submits unless guarded.
- Color constants from PLAN.md (`#ecad0a`, `#209dd7`, `#753991`) belong in one Tailwind theme definition.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Float arithmetic for cash and cost | Faster to write, matches REAL columns | Cent drift, dust positions, "-0.00" | Never for the trade path; fine for display-only derived numbers if rounded identically |
| Prices in React Context | Simple data flow | Re-render storm, flaky E2E | Never |
| Sync `litellm.completion` in handler | Copy-paste from skill | Server freezes during chat | Never in handlers |
| Loose `schema.sql` file in `db/` dir | Easy to read | Masked/frozen by volume | Never; embed in Python module |
| Skipping `.gitattributes` | One less file | CRLF breaks scripts/.env on Windows dev | Never on this repo |
| Test-only reset endpoint always on | Easy E2E isolation | Wipes user data in prod | Only if gated by an explicit env flag, off by default |
| One huge `main.py` | Fast start | Hard to test in isolation | Only for the first hour; use `create_app()` + routers from the start |
| Accepting any ticker in simulator | Meets "add PYPL" spec | LLM/user typos become tradeable fake stocks | Acceptable if documented and format-validated |
| No schema versioning | Zero migration code | Stale volumes break after schema edits | Acceptable if `user_version` check gives a clear "remove the volume" error |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| LiteLLM -> OpenRouter -> Cerebras | Sync `completion`, no timeout, `order` without `allow_fallbacks: false` | `acompletion`, `timeout`, strict provider routing, layered JSON parse |
| OpenRouter structured outputs | Assuming all endpoints honor strict `json_schema` | `require_parameters`, flat schema, validate in Python, fallback ladder |
| gpt-oss reasoning model | Low `max_tokens` -> empty `content` | Explicit `max_tokens`, `reasoning_effort="low"`, check `finish_reason` |
| Massive REST | No timeout, polling more than the tier allows | `asyncio.timeout`, interval from tier, only add tickers after a price comes back |
| EventSource | Per-component connections, "red on any error" | Singleton + readyState state machine + watchdog |
| Lightweight Charts | No `remove()`, duplicate/out-of-order times, SSR import | Cleanup in effect, per-second dedupe, `ssr:false` dynamic import |
| StaticFiles | Mount before routers, relative dir | Mount last, absolute path, existence guard |
| Docker `--env-file` | Quoted values, CRLF | Unquoted LF `.env`, strip/lower in code |
| Named volume at `/app/db` | Code also lives at `/app/db` | Keep code out of the mount point |
| Playwright in compose | Image/package version skew, no health wait | Pin both, `service_healthy`, `baseURL` to service name |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Full re-render per tick | High idle CPU, laggy typing | External store + selectors, memoized rows | Immediately at 10 tickers x 2Hz |
| Unbounded series arrays | Memory growth, slow charts | Ring buffer (300-600 pts) | After about 30-60 min open tab |
| 10 canvas charts for sparklines | Slow first paint | SVG polyline sparklines | At 10+ tickers |
| Blocking SQLite in handlers | SSE jitter on writes | aiosqlite or to_thread with lock, WAL | On slow volume fsync |
| Blocking LLM call | All clients freeze | `acompletion` | Every chat message |
| Treemap/P&L recompute every tick | Jank | Throttle to 1Hz / on trade | With several positions |
| Heartbeat-less SSE | Silent stale UI | `: ping` every 15s + client watchdog | With proxies or Massive 15s polling |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| `.env` copied into image | API key leaks in layers | `.dockerignore`, pass via `--env-file` at runtime only (MEDIUM severity) |
| Rendering LLM output as HTML | XSS from model/user text | Render as plain text (MEDIUM) |
| Unvalidated ticker / quantity from LLM and API | Garbage rows, NaN, oversized strings | Single Pydantic validation layer for both entry points (MEDIUM) |
| LLM treated as authority | Over-trading from hypotheticals or injected text in history | Explicit-intent prompt, trade cap per response, server-side validation identical to manual path (MEDIUM) |
| Test reset endpoint left enabled | Anyone can wipe state | Env-gated, off by default (LOW on localhost) |
| Logging API keys / full prompts at DEBUG | Secret leak in logs | Redact; do not log `Authorization` or env (LOW) |
| No input length limit on chat | Huge prompts, cost | Max length on request model (LOW) |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Status dot green while data is stale | Trading on frozen prices | Watchdog -> yellow |
| `null` price for new ticker | "NaN"/crash | "--" placeholder, row spinner |
| Chat text claims success on failed trade | Confusion about balance | Render from structured `actions`, show failures in red |
| Empty P&L chart/treemap on first launch | Looks broken | Seed snapshot, empty-state copy |
| Sparklines empty after load | Looks broken | Brief "collecting..." state; documented in spec as progressive fill |
| No loading state during LLM call | Double sends | Disable input, spinner |
| Selected ticker removed from watchlist | Blank main chart | Fall back to first ticker |
| Buy of a typo ticker "succeeds" | Fake position | Validation + clear error toast |

## "Looks Done But Isn't" Checklist

- [ ] **SSE:** works in `curl -N`, but verify through the final container with static mount, no gzip, `Content-Type: text/event-stream`, heartbeat present, and `docker stop` completes in under 5s.
- [ ] **Static serving:** `/api/health` returns JSON, `/` returns the app, `/api/nonexistent` returns JSON 404, refresh works.
- [ ] **Fresh volume:** `docker run` with a brand-new volume seeds $10,000 and 10 tickers with no errors; second run keeps data; removing the volume resets.
- [ ] **Trade math:** buy exact cash, sell all, partial sells, fractional quantities, negative/NaN/zero rejected, concurrent buys.
- [ ] **Watchlist removal with open position:** total value and heatmap unchanged.
- [ ] **Chat:** malformed JSON, `content=None`, failed trade, 5+ trades, hypothetical question, repeated "yes", provider outage, `LLM_MOCK` with no key and no litellm import.
- [ ] **Chat-driven changes** refresh watchlist, positions, header and chart without reload.
- [ ] **Reconnect:** kill backend, dot goes yellow/red, restart backend, dot goes green, prices resume without a page reload.
- [ ] **Flash:** only on actual price change; not on first render; not when a ticker is re-sent unchanged.
- [ ] **Memory:** 30 min tab soak, flat heap; ticker switching x50 leaves one chart canvas.
- [ ] **Windows:** `.gitattributes` in place, scripts run from a fresh clone on both mac/linux (LF) and PowerShell, `.env` with CRLF still works.
- [ ] **E2E:** green three runs in a row, with `--repeat-each=3` and with a randomized order.

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| CRLF in scripts/files | LOW | Add `.gitattributes`, `git add --renormalize .`, commit |
| Static mount shadowing API | LOW | Move mount to end of app construction |
| Re-render storm | HIGH if built on Context | Introduce external store; migrate selectors component by component |
| Float cash drift in stored data | MEDIUM | Round-on-write migration; recompute cash from trades table (trades are append-only, so reconstruct) |
| Schema frozen in volume | LOW | `docker volume rm finally-data`; move schema into Python module |
| Structured output instability | MEDIUM | Add fallback ladder (json_object + prompt schema + tolerant parse); pin litellm |
| Non-atomic trades | MEDIUM | Wrap in `BEGIN IMMEDIATE` transaction; reconcile cash from trade log |
| Flaky E2E from shared state | MEDIUM | Add gated reset endpoint, serialize workers |
| Chart memory leaks | LOW-MEDIUM | Add `remove()` cleanup, ring buffer |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| 1 StaticFiles shadow / missing dir | P1, P4 | Test: `/api/health` is JSON with static mounted; app boots without `static/` |
| 2 SSE cancel / shutdown / duplicate routes | P1 | Test: second `create_app()` has no duplicate routes; `docker stop` under 5s |
| 3 Wrong daily change % | P1 contract, P3 | Reload page: value persists; ticker with no tick shows non-zero day change |
| 4 Flash on stale direction | P3 | Unit test: identical price twice -> no flash |
| 5 Status dot semantics | P3, P5 | E2E reconnect scenario; stale watchdog unit test |
| 6 Re-render storm | P3 | Profiler: only changed rows re-render per tick |
| 7 Chart leaks / ordering / SSR | P3 | `next build` passes; 50 ticker switches leave one canvas; duplicate-second update test |
| 8 Float money / avg cost | P1 | Property/unit tests for buy/sell sequences; no dust rows |
| 9 Non-atomic trades | P1 | `gather` of two over-cash buys: exactly one succeeds |
| 10 SQLite async / WAL / init | P1 | Concurrent snapshot + trade test; fresh-volume boot test |
| 11 Watchlist/source/position coupling | P1 | Remove watchlist ticker with position: valuation unchanged |
| 12 Sync LiteLLM | P2 | Event-loop-responsive test during mocked slow LLM |
| 13 Structured output quirks | P2 | Live smoke test; parser unit tests for fences, None, bad enum |
| 14 Chat claims vs outcomes / duplicates | P2, P3 | Test: failed trade -> action status failed and UI shows red; history excludes executed-trade phrasing |
| 15 Volume masks code | P1, P4 | Rebuild image, change schema, fresh volume picks up change |
| 16 Windows CRLF etc. | P1 (first commit), P4 | Fresh clone on Windows: `file scripts/*.sh` shows LF; container run |
| 17 SSE / lifespan test traps | P1 | SSE generator tests exist and terminate |
| 18 Next.js export limits | P3 | `next build` with `dynamic='error'`; no `app/api` |
| 19 REST vs SSE drift, stale after chat | P3 | E2E: chat buy updates positions/header without reload |
| 20 Snapshot edge cases | P1, P3 | History non-empty at startup; same-second dedupe test |
| 21 Docker build issues | P4 | Smoke script: build, run, health, `/`, first SSE chunk |
| 22 Playwright flake | P2 (mock contract), P5 | 3 consecutive green runs, random order |

## Research Flags (for roadmap/phase planning)

- **P2 (LLM chat): needs a short spike.** Confirm with the real `OPENROUTER_API_KEY`: does `response_format` with a Pydantic model work through LiteLLM -> OpenRouter -> Cerebras gpt-oss-120b with `allow_fallbacks: False`? What does the actual schema sent look like (all-required)? What happens with `reasoning_effort="low"` and token limits? Output feeds the parser fallback design.
- **P5 (SSE reconnect E2E): needs a small spike.** Which technique reliably severs an open EventSource under Playwright in Docker (route abort, test-only stream-drop endpoint, or container pause)?
- **P3 (charts):** pin Lightweight Charts major version (v4 vs v5 API) and confirm the treemap approach (Recharts Treemap vs custom d3-hierarchy) at plan time. Standard patterns otherwise.
- **P1, P4:** standard patterns, no research needed beyond the items above.

## Sources

- Repo inspection (HIGH): `backend/app/market/stream.py`, `cache.py`, `simulator.py`, `factory.py`, `models.py`, `backend/pyproject.toml`, `.planning/codebase/CONCERNS.md`, `planning/PLAN.md`, `.claude/skills/cerebras/SKILL.md`; git config `core.autocrlf=true` and absence of `.gitattributes`/`.dockerignore`.
- OpenRouter structured outputs documentation: https://openrouter.ai/docs/guides/features/structured-outputs (HIGH: provider-dependent support, `require_parameters`, strict not guaranteed on all endpoints)
- Cerebras on OpenRouter: https://inference-docs.cerebras.ai/integrations/openrouter.md (MEDIUM: provider routing, structured output example)
- Next.js static exports guide (v16.2.9 docs via Context7): `docs/01-app/02-guides/static-exports.mdx` (HIGH: `dynamic = 'error'`, dynamic routes need `generateStaticParams`)
- Starlette StaticFiles docs: https://starlette.dev/staticfiles/ (HIGH: `html=True`, mount semantics)
- Established behaviour from training knowledge (MEDIUM): EventSource readyState semantics, Lightweight Charts time ordering and `remove()`, SQLite WAL/locking and Python `sqlite3` transaction handling, httpx ASGITransport buffering, Docker named-volume population and `--env-file` quoting, Playwright/Docker version pairing, LiteLLM import-time cost-map fetch. Items tagged **[VERIFY]** should be confirmed in a spike.

---
*Pitfalls research for: AI-powered simulated trading workstation (FinAlly)*
*Researched: 2026-10-27*
