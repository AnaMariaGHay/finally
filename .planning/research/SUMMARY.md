# Project Research Summary

**Project:** FinAlly — AI-Powered Trading Workstation
**Domain:** Real-time paper trading platform with agentic LLM copilot, single-user, simulated or real market data via REST polling
**Researched:** 2026-10-27
**Confidence:** HIGH overall

## Executive Summary

FinAlly is a full-stack single-container app (FastAPI + Next.js + SQLite + SSE + LiteLLM/OpenRouter) that combines Bloomberg-terminal UX, live-streaming prices, and an AI trading copilot that auto-executes trades. Experts build this pattern by decoupling concerns (market data, portfolio service, LLM chat) and using tight transaction boundaries to ensure atomicity.

**Recommended approach:** Prioritize backend platform stability (DB, trade logic) before UI. Isolate business logic in services so REST and LLM paths share validation. Use stdlib sqlite3 with per-operation transactions to avoid subtle concurrency bugs that plague async database libraries.

**Key risks:** (1) Concurrent trade atomicity under simultaneous manual + LLM requests — mitigated by `BEGIN IMMEDIATE` transactions and per-operation connections; (2) Event loop blocking from sync LLM calls — mitigated by async `acompletion`; (3) Schema code shadowed by Docker volumes — mitigated by embedding schema in Python modules; (4) Static file serving shadowing API routes — mitigated by registering all `/api` routers before the final mount. All are well-understood pitfalls with clear prevention patterns.

## Key Findings

### Recommended Stack

**Backend:** FastAPI 0.128.7, Uvicorn 0.40.0 (single worker), stdlib `sqlite3` with `asyncio.to_thread` (short per-operation transactions, WAL, busy_timeout, `BEGIN IMMEDIATE` per trade), LiteLLM 1.106.4+ with async `acompletion` (lazy-imported; never imported when `LLM_MOCK=true`), Pydantic 2.12+.

**Frontend:** Next.js 16.4.3 (static export), React 19.3.1, TypeScript 6.0.3 (pinned for eslint compatibility), Tailwind CSS 4.3.3, Zustand 5.0.16 (critical: not Context, to avoid render storms), TanStack Query 5.105.0, Lightweight Charts 5.2.1 (canvas, strictly ascending unique integer-second times), d3-hierarchy treemap, hand-rolled SVG sparklines.

**Tests:** pytest + httpx (backend); Vitest 5 + RTL (frontend); Playwright 1.64.1 with `mcr.microsoft.com/playwright:v1.64.1-noble` (E2E).

**Why:** Verified against registries 2026-10-27; chart library from maintainer docs; concurrency patterns from code inspection. See STACK.md for full detail.

### Expected Features

**Must have:** Live header (value + cash + status dot), watchlist with flash + daily change % + sparkline, main chart, heatmap treemap, P&L chart (seeded at startup), positions table, trade bar, AI chat panel with inline confirmations rendered from structured action results, SSE reconnect logic, empty/loading/error states.

**Should have:** Portfolio-aware analysis, starter prompts, average-cost line, cross-panel selection, visual drama from simulator.

**Defer:** Candlesticks, trade history view, token streaming.

**Anti-features:** Limit orders (spec: market only), confirmations, multi-user, short selling.

### Architecture Approach

Single container, one FastAPI worker, one SQLite connection per operation (not held across awaits), one background snapshot loop, one EventSource singleton per tab. Market data → PriceCache → SSE frames → Zustand store (per-ticker selectors) → UI. REST and LLM both call PortfolioService.execute_trade through identical validation. Chat pipeline: (1) build context, (2) call LLM, (3) parse/validate JSON, (4) execute watchlist changes then trades, (5) persist. No transaction held across the LLM call.

### Critical Pitfalls

1. **StaticFiles mount shadows `/api`** → Register routers first, mount static last; guard with `is_dir()`
2. **Concurrent trades corrupt cash** → Every trade in one `BEGIN IMMEDIATE` transaction; per-operation connections
3. **Daily change % from SSE (tick-to-tick)** → Store session-open baseline in PriceCache; expose `day_change_percent`
4. **Removed ticker's price orphans positions** → Track watchlist ∪ positions, not just watchlist
5. **Sync LiteLLM blocks event loop** → Use `await litellm.acompletion(...)`; set timeout; lazy-import
6. **Existing stream.py defects** → module-level router duplicates routes; swallowed `CancelledError` blocks shutdown
7. **Windows CRLF breaks .sh in Docker** → `.gitattributes` with `*.sh text eol=lf`; `.dockerignore`

## Implications for Roadmap

### Suggested 6 Phases (Coarse Granularity)

**Phase 1: Backend Foundation** — App entrypoint, DB schema, config, market extensions (reference_price), lifespan ordering, `/api/health`, SSE router fix, static mount stub. Exit: uvicorn streams prices with day_change_percent; DB seeded on fresh path; factory test harness.

**Phase 2: Portfolio & Watchlist APIs** — PortfolioService (trade math, atomicity), SnapshotService, WatchlistService (add/remove + reconcile_tracked), REST endpoints, validation. Exit: full REST works against simulator; portfolio math unit-tested; concurrent trade gather test.

**Phase 3: Chat Service & LLM** — MockLLMClient (keyword rules), LiteLLMClient (async acompletion, structured outputs), chat pipeline (context → LLM → parse → execute → persist), ActionResult schema, error handling, smoke test with real key. **Research spike:** Cerebras strict schema behavior.

**Phase 4: Frontend** — Next.js scaffold, SSE client singleton, Zustand store (selectors), all components (header, watchlist, main chart, heatmap, P&L, positions, trade bar, chat), tailwind theme. Exit: interactive app, all panels wired.

**Phase 5: Docker & Scripts** — Multi-stage Dockerfile, .dockerignore, named volume, healthcheck, start/stop scripts (mac + PowerShell), .gitattributes (LF for scripts). Exit: `docker build && docker run` on :8000 works.

**Phase 6: E2E & Hardening** — docker-compose.test.yml, Playwright scenarios (fresh start, buy, sell, chat, reconnect, flash), mock contract documented, test-only reset endpoint, verification checklist. Exit: 3 consecutive green runs.

### Phase Ordering Rationale

1. **P1 first:** everything depends on DB, factory, contracts
2. **P2 before frontend panels:** business logic + services provide REST contracts
3. **P3 parallel with P2 possible:** highest-risk integration; early spike validates LLM path
4. **P4 scaffold can start after P1:** SSE client needs only market extensions; panels wait for P2/P3 contracts
5. **P5 after all code:** packaging, reproducibility
6. **P6 final:** E2E, catchall verification

### Research Flags

- **P3 (LLM):** Confirm Cerebras strict json_schema with nested models (`$defs`/`$ref`) through OpenRouter. Small spike: one request with real key before full P3.
- **P6 (SSE reconnect E2E):** Which method reliably severs EventSource in Playwright (route abort, test endpoint, container pause)?
- **P4 (charts):** Lightweight Charts v5 API (`addSeries`/`update()`) — confirm at start.

Standard patterns (no research needed): P1, P2, P5, most P4 components.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Verified against npm/PyPI registries 2026-10-27; dependency compatibility chains checked |
| Features | MEDIUM | Grounded in established paper-trading UX; some spec gaps require implementation decision during planning |
| Architecture | HIGH | Derived from direct code inspection; concurrency patterns verified |
| Pitfalls | HIGH | Mapped to actual code, requirements, and ecosystem patterns |

**Overall:** HIGH confidence on path forward.

### Gaps to Address

- **Massive prev_day attribute:** Confirm exact name in `massive` package during P1
- **Cerebras nested schema:** Verify during P3 spike (live request)
- **EventSource reconnect in Playwright:** Test technique during P6
- **Lightweight Charts v5 vs v4:** Confirm API at P4 start
- **Windows path handling:** Test scripts on Windows during P5

## Deviations from PLAN.md (11 items)

1. **Node 24** (not 20; EOL, and Vitest 5/jsdom 30 need Node 22.12+)
2. **TypeScript 6.0.3 + ESLint 9.x** (peer constraints)
3. **Async acompletion** (not sync completion from skill snippet; blocks loop)
4. **d3-hierarchy treemap** (Recharts Treemap fallback)
5. **Schema in app/db/** (not backend/db/; volume shadowing + wheel packaging)
6. **Dev-only Next rewrites** (export mode forbids them)
7. **SQLite via stdlib + to_thread** (not aiosqlite; avoids cross-await transaction interleaving)
8. **GET /api/chat** (history endpoint, reload persistence)
9. **Startup snapshot** (P&L chart seeding)
10. **Reference price in PriceCache** (session-open baseline for daily change %)
11. **Reconcile tracked = watchlist ∪ positions** (pricing gap prevention)

## Sources

- STACK.md — npm/PyPI registries, official docs, Context7, local LiteLLM wire probe (2026-10-27)
- FEATURES.md — PLAN.md spec analysis, existing market code, paper-trading conventions, Context7 (Lightweight Charts)
- ARCHITECTURE.md — direct reading of `backend/app/market/`, MARKET_DATA_SUMMARY.md, PLAN.md
- PITFALLS.md — repo code + git config, OpenRouter structured outputs docs, ecosystem knowledge

---
*Research completed: 2026-10-27*
*Ready for roadmap: yes*
