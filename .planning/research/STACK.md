# Stack Research

**Domain:** AI-powered simulated trading workstation (FastAPI + SSE + SQLite backend, Next.js static-export frontend, LiteLLM chat, single Docker container)
**Researched:** 2026-10-27
**Scope:** Only the NOT-yet-built parts. The market data subsystem (`backend/app/market/`) is complete and is not re-researched.
**Overall Confidence:** HIGH on versions (all checked against the npm/PyPI registries on 2026-10-27), MEDIUM on three items flagged below (live Cerebras strict-schema behavior, treemap implementation choice, TypeScript pin).

Confidence legend: HIGH = verified from registry or official docs or a local probe; MEDIUM = verified from one source or reasoned from verified facts; LOW = unverified (none are used as a recommendation here).

---

## Recommended Stack

### Backend (additions to the existing uv project)

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| FastAPI | keep locked `0.128.7` (latest is 0.162.0) | App entrypoint, REST routers, lifespan, static mount | Already locked and the 73 market tests pass against it. Nothing in this milestone needs newer features. Do not bump just to bump; if a bump is wanted later, `uv lock --upgrade-package fastapi` and re-run the market suite. | HIGH |
| Uvicorn[standard] | keep `0.40.0` (latest 0.54.0) | ASGI server | Same reasoning. Run as `uvicorn app.main:app --host 0.0.0.0 --port 8000` with a single worker. The price cache, simulator, and snapshot task are in-process, so multiple workers would each run their own simulator. | HIGH |
| aiosqlite | `>=0.22.1` | Async SQLite access | See "DB decision" below. Thin wrapper over stdlib `sqlite3` (one dedicated thread per connection), so the event loop (SSE + simulator) is never blocked. Raw SQL fits 6 tiny tables. | HIGH |
| LiteLLM | `>=1.106.4,<2` (latest 1.106.4) | LLM call to OpenRouter -> Cerebras | Mandated by PLAN.md and the project `cerebras` skill. Verified locally that it emits a strict `json_schema` response_format when given a Pydantic class (see "LLM decision"). Requires Python `>=3.10,<3.15`, fine on 3.12. | HIGH |
| Pydantic | `>=2.12` (already transitive; latest 2.14.1) | Request/response models, LLM structured-output schema | The same model class is the LiteLLM `response_format` and the validator for the reply. | HIGH |
| python-dotenv | `>=1.2` | Load `.env` from project root in local (non-Docker) runs | LiteLLM already depends on it. In Docker the env arrives via `--env-file`, so loading must be tolerant of a missing file. | HIGH |
| sqlite3 (stdlib) | Python 3.12 | Schema creation script, sync test helpers | Schema DDL is plain SQL executed via aiosqlite; no extra dependency. | HIGH |

#### DB decision: aiosqlite + raw SQL (no ORM)

- **Use `aiosqlite` 0.22.1.** SQLModel (0.0.48, still pre-1.0) and SQLAlchemy 2.1 add a mapping layer, async-driver config, and Alembic-style thinking for 6 flat tables with a "lazy create + seed" requirement. Raw SQL with a `schema.sql` string is shorter and easier for agents to get right.
- **`sqlite3` in a threadpool** (`asyncio.to_thread`) is a valid alternative and has zero dependencies, but you end up hand-rolling the connection-thread discipline that aiosqlite already provides. Not worth it.
- **0.22 breaking change (must handle):** `Connection` no longer subclasses `Thread`. You MUST `await conn.close()` (or call `conn.stop()`) or the process hangs/warns at exit. Open the connection in the FastAPI `lifespan` and close it in the teardown. In pytest fixtures, close it explicitly too.
- **Connection config at open:** `PRAGMA journal_mode=WAL; PRAGMA foreign_keys=ON; PRAGMA busy_timeout=5000; PRAGMA synchronous=NORMAL;`, and open with `isolation_level=None` so you control transactions explicitly with `BEGIN IMMEDIATE`.
- **Concurrency rule (critical):** aiosqlite serializes individual statements on its thread, but it does NOT make a multi-statement transaction atomic across coroutines on one shared connection. Two concurrent trades (manual POST plus LLM-driven) would interleave their statements. Wrap every trade (read cash, validate, update positions, insert trade, update cash, insert snapshot) in a single module-level `asyncio.Lock` plus `BEGIN IMMEDIATE ... COMMIT`, or open a short-lived connection per write transaction. Recommended: one long-lived connection plus one write lock. Single user, so contention is nil.
- **Lazy init:** run `CREATE TABLE IF NOT EXISTS` for all 6 tables on lifespan startup and seed (`INSERT OR IGNORE`) the `default` user (cash 10000.0) and the 10 watchlist tickers. This satisfies "no migration step". Create the parent directory (`db/`) with `os.makedirs(exist_ok=True)` before connecting.
- **Money:** the spec uses REAL. Round computed values at the API boundary (`round(x, 2)` for display values, keep full precision for quantity and avg_cost) and compare cash with a small epsilon or round to cents before `cash >= cost` checks, otherwise "buy exactly all cash" fails on float noise.

#### LLM decision: LiteLLM `acompletion` + Pydantic `response_format` + OpenRouter provider routing

Verified by a local probe against LiteLLM 1.106.4 (request captured by a local fake server; no secrets used). Passing a Pydantic class produces this on the wire:

- `response_format = {"type": "json_schema", "json_schema": {"name": "<Class>", "strict": true, "schema": {..., "additionalProperties": false, "required": [all fields]}}}`
- `reasoning_effort: "low"`
- `provider: {"order": ["cerebras"]}` (from `extra_body`, merged into the top-level body)

Prescriptive usage (follows the project `cerebras` skill, with three deliberate changes):

```python
from litellm import acompletion          # NOT completion: the sync call blocks the event loop
MODEL = "openrouter/openai/gpt-oss-120b"
EXTRA_BODY = {"provider": {"order": ["cerebras"]}}   # optional hardening: add "require_parameters": True

resp = await acompletion(
    model=MODEL, messages=messages,
    response_format=ChatLLMResponse,      # Pydantic: message: str, trades: list[Trade], watchlist_changes: list[WL]
    reasoning_effort="low", extra_body=EXTRA_BODY,
    timeout=30, num_retries=1, max_tokens=2048,
)
parsed = ChatLLMResponse.model_validate_json(resp.choices[0].message.content)  # catch ValidationError -> graceful reply
```

1. **Use `acompletion` (async).** The skill's sync `completion` would freeze SSE streaming and the simulator for the duration of the call inside an `async def` route.
2. **Always `model_validate_json` and catch `ValidationError`/`JSONDecodeError`.** Treat a malformed or truncated reply as a normal chat error message (the spec already requires "graceful handling of malformed responses"). Set a generous `max_tokens` so reasoning tokens do not truncate the JSON.
3. **Lazy-import litellm inside the chat service**, and never import it when `LLM_MOCK=true`. LiteLLM's import is slow (seconds) and pulls boto3/tiktoken/huggingface-hub; lazy import keeps app startup, unit tests, and the E2E mock path fast.
4. `litellm.supports_response_schema("openrouter/openai/gpt-oss-120b")` returns `False` (model absent from LiteLLM's registry) but `response_format` and `reasoning_effort` are both in its supported OpenAI param list for the openrouter provider, and the probe shows they are forwarded. Do not gate on `supports_response_schema`.
5. **Provider routing:** `order: ["cerebras"]` alone still allows OpenRouter to fall back to other providers (`allow_fallbacks` defaults to true). That is acceptable for resilience; if a hard guarantee of Cerebras is wanted for demo speed, add `"allow_fallbacks": False`. `"require_parameters": True` makes OpenRouter skip providers that do not support `response_format`. Keep it as an easily toggled constant.

**Flagged risk (MEDIUM, needs a live smoke test in the LLM phase):** I could not run a live call (the `.env` key is a protected secret and was not read). Pydantic nested models emit `$defs`/`$ref` in the schema. Cerebras strict-mode support for `$ref` has not been verified. If the live call errors or ignores the schema, flatten the schema (inline the Trade and WatchlistChange objects) or inline refs before passing. First task of the LLM phase should be a 10-line live script that sends one structured request and validates it, before building the full chat flow.

**Supply-chain note (HIGH, verified by news search):** LiteLLM versions `1.82.7` and `1.82.8` on PyPI were malicious (published 2026-03-24, quarantined within hours). Current `1.106.x` is not affected. Mitigations: commit `uv.lock`, install with `uv sync --frozen` in the Dockerfile (never an unlocked `pip install litellm`), and add `litellm>=1.106.4` (the floor excludes the bad versions).

### Frontend (new `frontend/` project)

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| Next.js | `16.4.3` | Framework, static export | Current `latest` tag. `output: 'export'` is a first-class mode (docs updated 2026-08). Needs Node `>=20.9`. App Router, single client-rendered page. | HIGH |
| React / React DOM | `19.3.1` | UI | Required by Next 16. | HIGH |
| TypeScript | `~6.0.3` (NOT 7.x) | Typing | Latest is 7.0.2 (native Go compiler) and Next 16.4 does support it via the `tsc` CLI, BUT `typescript-eslint` 8.72.1 (pulled in by `eslint-config-next`) declares peer `typescript <6.1.0`, so TS 7 breaks the lint toolchain. 6.0.3 is the newest version every tool agrees on. Revisit when typescript-eslint supports 7. | MEDIUM |
| Tailwind CSS | `4.3.3` | Styling, dark theme | Current. v4 is CSS-first: no `tailwind.config.js`; custom palette lives in `@theme` in `globals.css`. | HIGH |
| @tailwindcss/postcss + postcss | `4.3.3` + `^8.5` | Tailwind build integration | The documented Next.js integration. | HIGH |
| lightweight-charts | `5.2.1` | Main ticker chart, P&L line/area chart | TradingView canvas library, v5 API. Real-time `series.update()` is exactly the SSE use case. Zero React coupling, tiny (one dep: fancy-canvas). | HIGH |
| d3-hierarchy | `3.1.2` (+ `@types/d3-hierarchy` 3.1.7) | Treemap layout math | Heatmap needs squarified treemap layout only; render the rectangles as plain absolutely-positioned divs (or SVG rects) colored by P&L. ~40 lines, no animation re-trigger quirks, trivial for Playwright to assert (`data-testid`, computed background color). | MEDIUM |
| zustand | `5.0.16` | Client store for the SSE price stream, selected ticker, connection state | High-frequency updates (10+ tickers every ~500ms) need selector-based subscriptions so only the changed row re-renders. Context would re-render the whole tree. | HIGH |
| @tanstack/react-query | `5.105.0` | REST state: portfolio, watchlist, history, chat | Gives cache invalidation after a trade or an LLM action (refetch portfolio plus history), loading/error states, and retries with little code. Optional; plain `fetch` plus zustand works if the team prefers fewer deps. | MEDIUM |
| clsx | `2.1.1` | Conditional class names (flash up/down classes) | Tiny. | HIGH |

#### Next.js static export config (prescriptive)

```ts
// frontend/next.config.ts
import type { NextConfig } from "next";
const isDev = process.env.NODE_ENV === "development";

const nextConfig: NextConfig = {
  // 'export' forbids rewrites/redirects/headers/proxy/Server Actions, so only enable it for builds
  ...(isDev ? {} : { output: "export" as const }),
  images: { unoptimized: true },        // default image loader is unsupported with export
  // trailingSlash left default: single page (/), emits out/index.html
  ...(isDev
    ? { async rewrites() { return [{ source: "/api/:path*", destination: "http://localhost:8000/api/:path*" }]; } }
    : {}),
};
export default nextConfig;
```

- Output goes to `frontend/out/`; the Dockerfile copies it to `/app/static` (or similar) and FastAPI mounts it.
- Everything is a Client Component driven from one `app/page.tsx`; no dynamic routes, no Route Handlers, no Server Actions.
- `rewrites()` is documented as unsupported with `output: 'export'`, hence the dev-only gating, which lets `npm run dev` on :3000 proxy `/api/*` (including the SSE stream) to a FastAPI on :8000. Watch that the dev proxy buffers SSE acceptably; if it does not, point the dev frontend straight at `http://localhost:8000` via the static build instead.
- Charts must be client-only: `'use client'`, create the chart once in `useEffect`, `chart.remove()` on cleanup, and import the chart components with `next/dynamic(..., { ssr: false })`.
- `next lint` no longer exists in Next 16; call `eslint .` directly with a flat `eslint.config.mjs` that imports `eslint-config-next`.

#### Chart implementation notes (decisions)

- **Main ticker chart and P&L chart: lightweight-charts v5.** v5 API only: `chart.addSeries(LineSeries | AreaSeries, opts)`; `addLineSeries`/`setMarkers` are v4 and do not exist. Use `autoSize: true`.
- **Time values are UTCTimestamp in SECONDS** and must be strictly ascending and unique per series. SSE ticks come every ~500ms, so two ticks can share a second. Use `series.update(point)` (it replaces when `time === last`) for live ticks, and de-duplicate by second before any `setData()` (history or P&L snapshots), or the library throws.
- **Sparklines: do NOT use a chart-library instance per row.** Hand-roll a ~30-line SVG `<polyline>` (or one small canvas) from a ring buffer (last ~120 points) in the zustand store. 10 to 20 chart instances each with their own canvases, resize observers, and 2 Hz updates is wasteful and gives nothing here.
- **Heatmap: d3-hierarchy `treemap().tile(treemapSquarify)`** over `{ticker, value = qty * price}` and color from a P&L scale (red to neutral to green, clamped at about +/-5%). Fallback if the team wants zero custom layout code: Recharts `3.10.1` `<Treemap>` with a custom `content` renderer and `isAnimationActive={false}` (Recharts is SVG, not canvas, but 10 rectangles is nothing). Both are acceptable; d3-hierarchy is recommended for deterministic E2E assertions.
- **Price flash:** compare new vs previous price in the store, set `data-flash="up|down"` (or a class) on the cell, and clear it after ~500ms. Style with a CSS transition on `background-color`. Tailwind v4 `@theme` holds `--color-up`, `--color-down`, accent `#ecad0a`, blue `#209dd7`, purple `#753991`.
- **SSE client:** native `EventSource('/api/stream/prices')`. Map `readyState`/`onopen`/`onerror` to the green/yellow/red dot (OPEN to green, CONNECTING after an error to yellow, CLOSED to red). EventSource auto-reconnects; do not wrap it in a library.

### Frontend testing

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| Vitest | `5.0.5` | Unit/component runner | Current. Needs Node `^22.12 || ^24 || >=26` and Vite `^6.4 || ^7 || ^8`. | HIGH |
| vite | `^8.3.5` | Vitest peer + plugin-react host | `@vitejs/plugin-react` 6.x requires `vite ^8`. Install it explicitly as a devDependency (Next does not use Vite). | HIGH |
| @vitejs/plugin-react | `6.1.2` | JSX transform in Vitest | Pairs with Vite 8. | HIGH |
| jsdom | `30.1.3` | DOM environment | Needs Node `^22.22.2 || ^24.15`. Dev host is Node 24.21, so fine. | HIGH |
| @testing-library/react | `16.3.3` | Component tests | Requires peer `@testing-library/dom ^10`, install both. Supports React 19. | HIGH |
| @testing-library/dom | `^10` | Peer of RTL 16 | Required peer. | HIGH |
| @testing-library/jest-dom | `7.0.1` | DOM matchers (`toHaveClass`, ...) | Needs Node `>=22`. Import `@testing-library/jest-dom/vitest` in the setup file. | HIGH |
| @testing-library/user-event | `14.6.8` | Realistic interactions (typing in trade bar, chat) | Standard. | HIGH |
| @vitest/coverage-v8 | `5.0.5` | Coverage (optional) | Version must match Vitest exactly. | HIGH |

Notes: jsdom cannot test CSS animation. Test the flash by asserting the class/data-attribute toggles on price change and clears after a timer (use `vi.useFakeTimers()`), not the visual fade. Mock `EventSource` with a tiny class in the Vitest setup file.

### Frontend lint

| Technology | Version | Purpose | Why | Confidence |
|------------|---------|---------|-----|------------|
| ESLint | `^9.39.5` (NOT 10.x) | Linting | Latest is 10.13.0, but `eslint-plugin-react@7.37.5` and `eslint-plugin-jsx-a11y` declare peers only up to ESLint 9, and both come in via `eslint-config-next`. Stay on 9.39 to avoid peer-dependency conflicts. | HIGH |
| eslint-config-next | `16.4.3` | Next rules (flat config) | Versioned in lockstep with Next. | HIGH |
| @types/node / @types/react / @types/react-dom | `^24` / `19.3.0` / `19.3.0` | Types | Match the Node major used in the Dockerfile. (`@types/node` latest is 26.x; use the major matching the runtime, 24.) | MEDIUM |

### E2E (new `test/` project)

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| @playwright/test | `1.64.1` | E2E runner (TypeScript) | Current. Engines `node >=20`. Lives in `test/package.json`, separate from `frontend/`. | HIGH |
| Docker image `mcr.microsoft.com/playwright` | `v1.64.1-noble` | Browser container in `docker-compose.test.yml` | The image tag MUST equal the `@playwright/test` version, otherwise the bundled browsers do not match. Tag verified to exist on MCR. Use it only for the runner, keeping browsers out of the prod image as PLAN.md requires. | HIGH |

`docker-compose.test.yml` shape (prescriptive):

- `app`: `build: ..` (root Dockerfile), `environment: LLM_MOCK=true`, no `OPENROUTER_API_KEY`, db on `tmpfs` or an anonymous volume so every run starts fresh, `healthcheck` that runs `python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/api/health')"` (the slim image has no curl).
- `playwright`: `image: mcr.microsoft.com/playwright:v1.64.1-noble`, `depends_on: app: {condition: service_healthy}`, `working_dir: /work`, volume-mount `./` to `/work`, `environment: BASE_URL=http://app:8000`, `init: true`, `ipc: host` (recommended by Playwright for Chromium), command `sh -c "npm ci && npx playwright test"`.
- Run Chromium only (one project) to keep CI quick. Use `retries: 1`, `trace: 'on-first-retry'`, `workers: 1` (the app is a stateful single user; parallel tests would trample each other's portfolio).
- Tests must be order-aware or reset state: there is a single `default` user. Either add a test-only reset path (see PITFALLS) or give each spec a fresh container. Prefer deterministic assertions on relative changes (cash decreases by price*qty) because simulator prices are random.

### Infrastructure (Docker)

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| Node build stage image | `node:24-slim` (deviates from PLAN.md's "Node 20") | `npm ci && npm run build` | Node 20 reached end-of-life in April 2026. Next 16 supports 20.9+, but Vitest 5 / jsdom 30 / jest-dom 7 require Node 22.12+/24, and the dev host runs 24.21. Use one major everywhere (add `.nvmrc` = 24). `node:22-slim` is also acceptable. | HIGH |
| Python runtime image | `python:3.12-slim` | Run FastAPI | Matches `requires-python >=3.12` and PLAN.md. | HIGH |
| uv | copy from `ghcr.io/astral-sh/uv:0.12` (host has 0.12.17) | `uv sync --frozen --no-dev` | `COPY --from=ghcr.io/astral-sh/uv:0.12 /uv /usr/local/bin/uv`. Use `--frozen` so the lockfile is authoritative (also the LiteLLM supply-chain mitigation). Set `UV_COMPILE_BYTECODE=1`, `UV_LINK_MODE=copy`. | HIGH |
| Docker named volume | `finally-data:/app/db` | SQLite persistence | Per PLAN.md. WAL mode creates `finally.db-wal` and `-shm` beside the DB, so mount the directory, never the single file. | HIGH |

Dockerfile outline: stage 1 `node:24-slim` copies `frontend/` and runs `npm ci && npm run build` (output `out/`). Stage 2 `python:3.12-slim` copies uv, copies `backend/pyproject.toml` + `uv.lock` first (layer cache), `uv sync --frozen --no-dev`, copies `backend/app`, copies `--from=stage1 /app/out` to `/app/static`, `EXPOSE 8000`, `CMD ["uv","run","--no-sync","uvicorn","app.main:app","--host","0.0.0.0","--port","8000"]`. Add a `.dockerignore` (`node_modules`, `.next`, `.venv`, `db/*.db*`, `.env`, `.git`, `.planning`).

FastAPI static serving: register all `/api/*` routers FIRST, then `app.mount("/", StaticFiles(directory=STATIC_DIR, html=True), name="static")` last, guarded by `if STATIC_DIR.is_dir()` so local dev and unit tests without a built frontend do not crash at import.

### Supporting Libraries (backend dev)

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| httpx | `>=0.28.1` | Starlette `TestClient` / `AsyncClient(transport=ASGITransport(app))` for API tests | Add to the `dev` extra. Needed for route tests (status codes, response shapes). Verify `TestClient` imports on first test (Starlette 1.x also supports `httpx2`). |
| pytest-asyncio | keep locked `1.3.0` (latest 1.4.1) | Async DB/route tests | Already configured `asyncio_mode = "auto"`. |
| pytest, pytest-cov, ruff | keep locked | Existing | No change. |

---

## Installation

```bash
# Backend (from backend/)
uv add litellm aiosqlite python-dotenv
uv add --optional dev httpx          # route tests

# Frontend (from frontend/), Node 24
npm install next@16.4.3 react@19.3.1 react-dom@19.3.1 \
  lightweight-charts@5.2.1 d3-hierarchy@3.1.2 zustand@5.0.16 \
  @tanstack/react-query@5.105.0 clsx@2.1.1 \
  tailwindcss@4.3.3 @tailwindcss/postcss@4.3.3 postcss
npm install -D typescript@~6.0.3 @types/node@^24 @types/react@19.3.0 @types/react-dom@19.3.0 \
  @types/d3-hierarchy@3.1.7 \
  eslint@^9.39.5 eslint-config-next@16.4.3 \
  vitest@5.0.5 @vitest/coverage-v8@5.0.5 vite@^8.3.5 @vitejs/plugin-react@6.1.2 jsdom@30.1.3 \
  @testing-library/react@16.3.3 @testing-library/dom@^10 \
  @testing-library/jest-dom@7.0.1 @testing-library/user-event@14.6.8

# E2E (from test/)
npm install -D @playwright/test@1.64.1       # must equal the playwright image tag
```

`postcss.config.mjs`: `export default { plugins: { "@tailwindcss/postcss": {} } };`  and `app/globals.css`: `@import "tailwindcss";` followed by an `@theme { --color-accent: #ecad0a; --color-primary: #209dd7; --color-submit: #753991; --color-surface: #0d1117; ... }` block.

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| aiosqlite + raw SQL | SQLModel 0.0.48 / SQLAlchemy 2.1 async | Only if the app grows to many tables, migrations, or a real multi-user Postgres move. Not for 6 flat tables. |
| aiosqlite + raw SQL | stdlib `sqlite3` + `asyncio.to_thread` | If you want zero new dependencies and accept hand-managing thread safety. |
| LiteLLM (mandated) | `openai` SDK pointed at `https://openrouter.ai/api/v1` | Much smaller dependency surface (LiteLLM drags in boto3, tiktoken, huggingface-hub, aiohttp) and avoids the LiteLLM supply-chain exposure. Not chosen because PLAN.md and the project skill mandate LiteLLM. Worth knowing if image size or security posture becomes a concern. |
| lightweight-charts 5 | Recharts 3.10.1 for price/P&L | Recharts is SVG and fine for static dashboards, but a tick-streaming terminal wants canvas and the incremental `update()` API. Keep Recharts only as the optional treemap fallback. |
| d3-hierarchy treemap | Recharts `<Treemap>` | Choose if you prefer zero layout code and accept SVG plus Recharts' animation/tooltip quirks. |
| zustand | React Context / Redux Toolkit | Context re-renders everything at 2 Hz; RTK is heavier than needed. |
| Vitest 5 + RTL | Jest | Jest needs extra Next/SWC config; Vitest works with the same Vite config style and is the current standard. |
| jsdom | happy-dom 21 | happy-dom is faster but has more edge-case gaps with RTL queries. Choose it only if tests get slow. |
| Next.js 16 static export | Vite + React SPA | Technically simpler for a pure SPA, but PLAN.md fixes Next.js; the course intent includes it. Do not switch. |
| Node 24 build stage | Node 20 (PLAN.md) | Only if a platform forces it; it is EOL and incompatible with the test toolchain. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Sync `litellm.completion` inside `async def` routes | Blocks the single event loop: SSE and simulator freeze during every LLM call (the cerebras skill snippet uses the sync call). | `await litellm.acompletion(...)` |
| `lightweight-charts` v4 API (`addLineSeries`, `series.setMarkers`, chart-level `watermark`) | Removed in v5; training-data examples are mostly v4. | `chart.addSeries(LineSeries, ...)`, `createSeriesMarkers` |
| `Date.now()` (milliseconds) as chart time | LWC expects UTCTimestamp in seconds; ms values render a nonsense far-future axis. | `Math.floor(Date.now()/1000)` |
| `setData()` on every SSE tick | Replaces the whole dataset and resets the visible range; very wasteful at 2 Hz. | `series.update(point)` |
| TypeScript 7.x for now | `typescript-eslint` 8.72.1 peer range is `<6.1.0`; lint toolchain breaks. | `typescript@~6.0.3` |
| ESLint 10.x for now | `eslint-plugin-react` 7.37.5 and `eslint-plugin-jsx-a11y` peers stop at ESLint 9. | `eslint@^9.39.5` |
| `next.config` `rewrites`/`redirects`/`headers` while `output: 'export'` | Documented unsupported; errors in `next dev` and build. | Gate dev-only rewrites on `NODE_ENV === "development"` and never enable `output: 'export'` in dev. |
| `next/image` default loader, Server Actions, Route Handlers using the request, dynamic routes without `generateStaticParams` | Unsupported in static export. | `images.unoptimized: true`, plain `fetch('/api/...')`, single page. |
| `next lint` | Removed in Next 16. | `eslint .` with flat config |
| LiteLLM 1.82.7 / 1.82.8 | Malicious releases (supply-chain attack, March 2026). | `litellm>=1.106.4` plus committed `uv.lock` and `uv sync --frozen` |
| `uvicorn --workers N` (N>1) | Each worker would run its own simulator, price cache, snapshot task; clients would see inconsistent prices. | Single worker |
| One chart-library instance per watchlist sparkline | 10+ canvases and observers for 30-px graphics. | SVG polyline from a ring buffer |
| Mounting the single `finally.db` file as a Docker volume | WAL sidecar files (`-wal`, `-shm`) live next to it and are lost/corrupt. | Mount the `db/` directory |
| Leaving `StaticFiles` mount before API routers | The `/` mount shadows later routes. | Include routers first, mount static last |
| Next.js `experimental.useTypeScriptCli = false` or `ignoreBuildErrors: true` as a shortcut | Hides real type errors. | Fix the types; run `tsc --noEmit` in CI |
| Shell scripts committed with CRLF | Dev host is Windows; `start_mac.sh` and Docker entrypoints break with `\r`. | `.gitattributes`: `*.sh text eol=lf` |

## Stack Patterns by Variant

**If developing the frontend interactively (HMR):**
- Run FastAPI on :8000 (`uv run uvicorn app.main:app --reload`) and `next dev` on :3000 with the dev-only `/api/*` rewrite.
- Because rewrites are not allowed under `output: 'export'`, the config must key off `NODE_ENV`.

**If testing the real artifact (E2E, demo):**
- Build with `docker build` and hit `http://localhost:8000`. Same origin, no CORS code anywhere.

**If `LLM_MOCK=true`:**
- Never import litellm. Return a deterministic mock keyed on message content (e.g. a message containing "buy" returns `trades: [{ticker: "AAPL", side: "buy", quantity: 1}]`), so Playwright can assert an inline trade confirmation without any API key.

**If `MASSIVE_API_KEY` is set:**
- Existing market subsystem handles it; the new watchlist API just calls the existing source's add/remove-ticker hooks. No new library choices.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| next@16.4.3 | react@19.3.1, node `>=20.9` | Verified from `engines`/`peerDependencies`. |
| next@16.4.3 | typescript 6.0.x (default checker) and 7.x (via `tsc` CLI) | Both work for `next build`; only 6.0.x also works with typescript-eslint. |
| eslint-config-next@16.4.3 | typescript-eslint ^8.56 (resolves to 8.72.1, peer `typescript <6.1.0`) | Drives the TS 6.0.x pin. |
| eslint@9.39.x | eslint-plugin-react@7.37.5, eslint-plugin-jsx-a11y, eslint-plugin-react-hooks 7.1 | ESLint 10 is outside react/jsx-a11y peer ranges. |
| vitest@5.0.5 | node `^22.12 || ^24 || >=26`, vite `^6.4 || ^7 || ^8` | Node 20 cannot run the test suite. |
| @vitejs/plugin-react@6.1.2 | vite `^8` only | Install `vite@^8.3.5` explicitly. |
| jsdom@30.1.3 | node `^22.22.2 || ^24.15 || >=26` | Dev host Node 24.21 OK. |
| @testing-library/jest-dom@7.0.1 | node `>=22` | |
| @testing-library/react@16.3.3 | @testing-library/dom ^10 (peer), react 18/19 | |
| tailwindcss@4.3.3 | @tailwindcss/postcss@4.3.3 (must match) | Pin both to the same version. |
| lightweight-charts@5.2.1 | fancy-canvas 2.1.0 (bundled dep) | `CanvasRenderingTarget2D` imports come from `fancy-canvas` for plugins. |
| @playwright/test@1.64.1 | `mcr.microsoft.com/playwright:v1.64.1-noble` | Exact match required between package and image tag. |
| litellm@1.106.4 | Python `>=3.10,<3.15`, pydantic `>=2.11`, openai `>=2.20` | Works on the project's Python 3.12. |
| aiosqlite@0.22.1 | Python `>=3.9` | Must explicitly `await close()`/`stop()`; no longer a Thread subclass. |
| fastapi 0.128.7 (locked) | starlette 0.52.1, pydantic 2.12.5 | Existing lock; no change needed. |

## Deviations from PLAN.md (call out in the roadmap)

1. **Node 24 (not Node 20)** for the build stage and local dev. Node 20 is EOL and cannot run the Vitest 5 toolchain.
2. **TypeScript pinned to 6.0.x and ESLint to 9.x** even though 7.x / 10.x are newer, for peer-dependency reasons.
3. **`acompletion` instead of the skill's sync `completion`.**
4. **Treemap via d3-hierarchy** rather than a charting library (Recharts `<Treemap>` stays as an approved fallback).
5. **Dev-only Next rewrites** (config must branch on `NODE_ENV`) because `output: 'export'` forbids them.

## Sources

- npm registry (`npm view`, 2026-10-27): next 16.4.3, react 19.3.1, tailwindcss 4.3.3, typescript 7.0.2 (6.0.3 chosen), lightweight-charts 5.2.1, recharts 3.10.1, vitest 5.0.5, @vitejs/plugin-react 6.1.2, vite 8.3.5, jsdom 30.1.3, @testing-library/* versions, @playwright/test 1.64.1, eslint 10.13.0 / 9.39.5, zustand 5.0.16, @tanstack/react-query 5.105.0, d3-hierarchy 3.1.2; engines/peerDependencies read directly. HIGH.
- PyPI JSON API (2026-10-27): litellm 1.106.4, aiosqlite 0.22.1, fastapi 0.162.0, uvicorn 0.54.0, pydantic 2.14.1, sqlmodel 0.0.48, sqlalchemy 2.1.6, pytest-asyncio 1.4.1, httpx 0.28.1; locked versions from `backend/uv.lock`. HIGH.
- Context7 `/tradingview/lightweight-charts/v5.2.1` (v5 `addSeries` API, seconds-based UTCTimestamp, `update()` semantics, Next.js client-only guidance). HIGH.
- Context7 `/vercel/next.js/v16.2.9` plus https://nextjs.org/docs/app/guides/static-exports (updated 2026-08-09): static export config and unsupported-feature list. HIGH.
- https://nextjs.org/docs/app/api-reference/config/typescript (updated 2026-08-25): TypeScript 7 support via `tsc` CLI. HIGH.
- https://tailwindcss.com/docs/installation/framework-guides/nextjs: Tailwind 4 + Next.js PostCSS setup. HIGH.
- https://openrouter.ai/docs/features/provider-routing: `order`, `allow_fallbacks`, `require_parameters`, `only`. HIGH.
- https://docs.litellm.ai/docs/completion/json_mode: Pydantic class as `response_format`, `supports_response_schema`. HIGH.
- Local probe, `litellm==1.106.4` against a loopback fake server: confirmed `response_format` json_schema (strict, `additionalProperties:false`), `reasoning_effort`, and `provider.order` on the wire; `supports_response_schema` False for this model. HIGH.
- https://github.com/omnilib/aiosqlite/blob/main/CHANGELOG.md: 0.22.0 close/stop requirement, 0.22.1 `stop()`. HIGH.
- MCR tag list (`mcr.microsoft.com/v2/playwright/tags/list`): `v1.64.1-noble` exists. HIGH.
- Web search (Netspi, Comet, Arthur, Hackernoon et al.): LiteLLM 1.82.7/1.82.8 compromise, 2026-03-24. MEDIUM-HIGH (multiple concordant secondary reports).
- Web search (achromatic.dev, Next.js docs): Next 16.3+ TS 7 support. MEDIUM-HIGH, confirmed by the official docs page above.
- NOT verified: live Cerebras behavior with nested `$defs` strict schemas (no API key access by design). Flagged for an early smoke test.

---
*Stack research for: AI trading workstation (remaining platform)*
*Researched: 2026-10-27*
