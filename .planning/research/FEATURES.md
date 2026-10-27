# Feature Research

**Domain:** AI-assisted simulated (paper) trading workstation, Bloomberg-terminal aesthetic, single user, market orders only
**Researched:** 2026-10-27
**Confidence:** MEDIUM overall. Feature expectations come from established paper-trading and terminal UX conventions (Robinhood, thinkorswim paperMoney, TradingView paper trading, Finviz map, Investopedia simulator, Bloomberg) as known to the researcher. They were not re-verified against live product pages this session. Codebase facts (marked "verified in code") and the Lightweight Charts time-ordering rule (Context7 maintainer docs) are HIGH.

Scope is fixed to `planning/PLAN.md`. This document does not expand scope. It says what "good" looks like inside each PLAN.md feature, what to skip, and where the spec is silent or ambiguous (see "Spec Gaps and Ambiguities"; resolve these during requirements).

## Facts Verified in the Existing Codebase (they constrain features)

| Fact | Source | Consequence |
|------|--------|-------------|
| SSE sends ONE event containing a dict of all changed tickers: `data: {"AAPL": {...}, ...}` | `backend/app/market/stream.py`, `backend/CLAUDE.md` | Frontend parses a map per event, not one event per ticker. Event cadence is version-based, so unchanged data produces no event (quiet stream is normal, especially with Massive polling at 15s). |
| `PriceUpdate.previous_price` and `change_percent` are tick-over-tick, NOT versus session open | `backend/CLAUDE.md` | The "daily change %" in the watchlist cannot use `change_percent`. A session-open baseline must be exposed separately (PROJECT.md decision). |
| Simulator accepts ANY ticker string; unknown ones get `random.uniform(50, 300)` as a seed price | `simulator.py` `_add_ticker_internal` | No built-in ticker validation. `FOO` or an LLM-hallucinated symbol "works" with a fake price. Validation must be a deliberate feature. |
| `SimulatorDataSource.add_ticker` seeds the cache immediately (price available instantly) | `simulator.py` | In simulator mode a new watchlist row has a price at once. With Massive the first price can take up to one poll interval (15s on the free tier). |
| `remove_ticker` calls `cache.remove(ticker)`, so the price disappears | `simulator.py` | Removing a watchlist ticker that has an open position would make that position unpriceable (portfolio value, heatmap, and sells break). See Watchlist edge cases. |
| `cache.get_price` returns `None` for unknown tickers | `backend/CLAUDE.md` | Trade execution must handle "no price available" explicitly. |

---

## Feature Landscape

### Table Stakes (Users Expect These)

Missing any of these makes the demo feel broken or unfinished. Items are grouped by PLAN.md feature.

#### 1. Header (total value, cash, connection status)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Total portfolio value that ticks live | Every terminal and brokerage shows live equity. A static number kills the "live" feel. | LOW | Compute client-side as `cash + sum(qty * livePrice)` from the last `/api/portfolio` fetch plus SSE prices. Do not poll the backend every 500ms. Re-fetch `/api/portfolio` after every trade or chat action to resync quantities and cash. |
| Cash balance | Spec requirement and the basic "buying power" number for a cash-only account. | LOW | Updates only on trades. |
| Connection dot green/yellow/red | Spec requirement. Users distrust prices without it. | MEDIUM | Map `EventSource.readyState`: OPEN is green, CONNECTING (initial, or auto-retrying after a drop) is yellow, CLOSED is red. Pitfall: when the server answers with a non-200 or wrong content-type, the browser sets CLOSED and does NOT retry. The client must run its own reconnect with backoff for that case. Add a text label or tooltip ("Live", "Reconnecting", "Offline") so color is not the only signal. |
| Total P&L versus the $10,000 start, in $ and % | A trader's first question is "am I up?" The header number alone does not answer it. | LOW | Derivable with no schema change: `total_value - 10000`. Treat 10000 as a constant, because there is no stored starting-capital column. Listed as table stakes because it is nearly free and the P&L story depends on it. |
| Sensible values before data arrives | The first paint must not show `NaN`, `$0.00`, or an empty bar. | LOW | Show `$10,000.00` from the initial `/api/portfolio` fetch (REST, not SSE). Show a skeleton or `--` only until that fetch resolves. |

#### 2. Watchlist panel (flashing prices, daily change %, sparklines, add/remove)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Row layout: ticker, price, daily change %, sparkline | Spec requirement and the universal watchlist layout. | MEDIUM | Use monospace or `tabular-nums` so digits do not jitter. Right-align numbers. Fixed 2-decimal prices. |
| Green/red flash (~500ms fade) on price change | Spec's signature visual. | MEDIUM | Compare the new price to the last client-known price for that ticker. Flash only on a real change and NEVER on the initial population (otherwise all 10 rows flash on load). Re-trigger when ticks arrive faster than the fade (remove the class, force reflow or use a key, re-add it). Honor `prefers-reduced-motion`. |
| Stable row order | Rows that re-sort on every tick are unusable. | LOW | Order by `added_at` (server order). No sorting by change in v1. |
| Click row to select (highlighted) | Spec: clicking selects the main chart ticker. | LOW | Also pre-fill the trade bar ticker with the selection (cheap, expected). |
| Add ticker input with inline error feedback | Spec: manual add. | MEDIUM | See ticker validation below. |
| Remove ticker (hover "x") | Spec: manual remove. | LOW | No confirm dialog. If the removed ticker was selected, fall back to the first remaining ticker, or to an empty state if none are left. |
| Pre-first-tick state | The user sees the page before any SSE event. | LOW | Do NOT wait for SSE. Initial render comes from `GET /api/watchlist` (it returns latest prices; the cache is seeded at start). Show `--` for price/change only when a row truly has no price yet (e.g. a just-added Massive ticker). The sparkline shows an empty or flat placeholder until it has 2 or more points. |
| Held-position marker (optional dot or bold) | Not required, but an obvious aid. | LOW | Skip if time is tight. |
| Empty watchlist state | User can remove all 10. | LOW | "Add a ticker to start" message. The app must not crash and the chart shows "Select a ticker". |

Sparkline expectations:
- Accumulated since page load only (spec). A ring buffer of roughly 120 to 300 points per ticker is plenty (about 1 to 2.5 minutes at 500ms).
- The buffer must live in a global store keyed by ticker, NOT inside the row component. Otherwise switching or re-mounting loses history, and the main chart cannot reuse it.
- Buffer ALL watchlist tickers, not just the selected one. The user must see an already-filled chart when they click a ticker.
- Normalize the y-axis to the buffer min/max. A sparkline scaled from 0 looks flat.
- Color the line by direction over the buffer (last vs first), or neutral blue.

Daily change % expectations:
- Baseline is the session-open price (simulator) or previous close (Massive), per PROJECT.md.
- The SSE payload does not carry that baseline. Expose it via `GET /api/watchlist` (e.g. `open_price` or `day_change_pct` at fetch time) and have the client compute `(livePrice - open) / open`. Flagged below as a spec gap.
- A ticker added mid-session gets a baseline equal to its price at add time (change shows +0.00% initially). That is correct and must not be treated as a bug.
- A backend restart resets simulator baselines. Expected, and worth a line in the README.

Ticker validation when adding (decision needed; recommendation given):
- Normalize: trim, uppercase.
- Format check (backend, 4xx with message): `^[A-Z]{1,5}([.-][A-Z]{1,2})?$` (allows `BRK.B`). Reject empty, spaces, and symbols that are too long.
- Duplicate: idempotent success or a gentle "already on your watchlist", not a red error. The DB has `UNIQUE(user_id, ticker)`, so a naive insert would 500. Catch it.
- Existence: with Massive, verify via a snapshot call at add time and reject if there is no data. With the simulator, accept any well-formed symbol and document that unknown symbols get simulated prices. If this feels wrong for the demo, add a small known-symbol allowlist (50 to 100 liquid tickers) as the cheapest honest alternative. Do not build a symbol-search service.
- Unknown ticker in LLM `watchlist_changes` goes through the same validator and reports failure in the action result.

#### 3. Main chart (selected ticker)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Price-over-time line or area chart that grows live | Spec minimum. | MEDIUM | Lightweight Charts: `series.update(point)` for ticks; `setData` only for initial seeding from the buffer. Chart code must be client-only (`'use client'`, `next/dynamic(..., { ssr: false })`) because of the Next.js static export. |
| Header strip: symbol, last price, daily change | Without it the chart has no context. | LOW | Reuse the watchlist's live data. |
| Crosshair and tooltip showing price and time | Basic chart interactivity. | LOW | Built in to Lightweight Charts. |
| Seeded from the existing buffer on ticker switch | Switching should not show an empty chart. | LOW | Depends on the global per-ticker buffer. |
| "Collecting data..." state while fewer than 2 points | A chart is empty for the first second of the session. | LOW | Chart axes render; text overlay. No spinner needed. |

Critical edge case (HIGH confidence, Context7 maintainer docs): Lightweight Charts requires series data STRICTLY ASCENDING by time with UNIQUE integer-second `UTCTimestamp` values. `update()` appends when `time > last` and replaces when `time === last`; earlier times throw. SSE ticks arrive every ~500ms, so two ticks share one second. The client must floor timestamps to seconds and collapse duplicates (last value wins), and must guard against out-of-order or clock-skewed backend timestamps. The same rule applies to the P&L chart.

#### 4. Portfolio heatmap (treemap)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Rectangle per position, area proportional to market value | Spec. Finviz-style convention. | MEDIUM | Lightweight Charts has no treemap. Use Recharts `Treemap` with custom cell content, or a small hand-rolled squarified layout. |
| Color by unrealized P&L % (green profit, red loss) | Spec. | LOW | Use a diverging scale with intensity, not a binary color. Saturate at a small threshold (about +/-3%) because simulated moves are small; with a +/-20% scale everything looks grey. Near-zero is neutral dark grey, not green or red. |
| Label: ticker and P&L % inside each cell | Without labels a heatmap is decoration. | LOW | Hide the label when the cell is too small. Tooltip on hover with qty, value, P&L $. |
| Empty-portfolio state | Fresh start has zero positions, and the e2e "fresh start" test hits this. | LOW | Placeholder panel: "No positions yet. Buy a stock to see your portfolio map." Never render an empty or broken SVG. |
| Single-position case | One position should fill the whole panel. | LOW | Natural for a treemap, but test it. |
| Re-render without jitter | Sizes change slightly on every tick. At 2 updates per second, layout shifts are distracting. | MEDIUM | Throttle heatmap updates to about 1 to 2s, and disable layout animations. |

Decision: should cash be a cell? The spec says "positions sized by weight". Recommend positions only; show a "Cash x%" label elsewhere (header). Keep cash out of the treemap.

#### 5. P&L chart (portfolio value over time)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Line of total portfolio value from `portfolio_snapshots` | Spec. | MEDIUM | `GET /api/portfolio/history`. Time axis. |
| Horizontal reference line at $10,000 | Gives "up or down versus start" at a glance. | LOW | Lightweight Charts price line, or a baseline series with a green-above/red-below fill. |
| Not empty on a fresh start | The e2e test expects "P&L chart has data points" and a fresh user would otherwise see a blank panel. | LOW | Backend must write one snapshot at startup or on first request (seed), in addition to the 30s task and post-trade snapshots. |
| Refresh after trades | A post-trade snapshot is only useful if the chart shows it. | LOW | Re-fetch history on trade or chat action, plus on a ~30s timer. |
| "Collecting data..." state with fewer than 2 points | Honest empty state. | LOW | Same pattern as the main chart. |
| Dedupe timestamps | A trade snapshot and the 30s snapshot can land within the same second; Lightweight Charts throws on duplicates. | LOW | Collapse by second, last value wins. |

Edge cases:
- A trade at the current price with no fees does not change total value, so the post-trade snapshot is mostly a duplicate point. It is still spec-required; just ensure it does not break the chart.
- Container downtime leaves gaps; the line interpolates across them. Acceptable, document it.
- Growth: 30s cadence is about 2,880 rows/day. Fine for SQLite. Cap or downsample the response (e.g. last 24h or N points) only if the chart slows; not needed for v1.
- Do NOT append a live point on every SSE tick to the chart. The header already shows live value.

#### 6. Positions table

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Columns per spec: ticker, qty, avg cost, current price, unrealized P&L ($), % change | Spec. | LOW | Current price and P&L update live from SSE. |
| Color-coded P&L (green/red, with sign) | Universal. | LOW | `+$12.40` and `-$3.10`, not just color (accessibility). |
| Empty state ("No open positions") | Fresh start. | LOW | |
| Fractional quantity display | Fractional shares supported. | LOW | Show up to 4 decimals with trailing zeros trimmed: `10`, `2.5`, `0.3333`. |
| Position disappears when fully sold | Spec behavior ("position updates or disappears"). | LOW | Backend deletes the row when quantity is effectively zero (see P&L math). |
| Default sort by market value descending | Biggest positions on top. | LOW | |

Ambiguity: "% change" is not defined in the spec. Recommend unrealized P&L % versus average cost (`price / avg_cost - 1`), the same number the heatmap color uses. The watchlist already has daily change %.

#### 7. Trade bar (market orders)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Ticker, quantity, Buy, Sell | Spec. | LOW | Uppercase ticker on input. Quantity accepts decimals. |
| Immediate, visible result feedback | Spec removes the confirmation dialog, so feedback after the fact becomes essential. | LOW | Inline success line ("Bought 5 AAPL @ $190.12") and inline error line near the bar (or a toast). Auto-clear after a few seconds. |
| Disable buttons while a request is in flight | Prevents double-submit duplicate trades. | LOW | |
| UI refresh right after a fill | The user expects cash, positions, heatmap, and header to react instantly. | LOW | Re-fetch `/api/portfolio` (and history) on success; do not wait for a poll. |
| Clear, specific rejections | Spec validation: insufficient cash, insufficient shares. | LOW | Messages with numbers: "Insufficient cash: need $1,901.20, have $1,500.00"; "You hold 3 AAPL, cannot sell 5." |
| Input validation | Zero, negative, non-numeric, NaN, Infinity, empty ticker, or an unpriced ticker. | LOW | Validate on the client for UX and on the server for correctness (server is authoritative). |
| Pre-fill from selected ticker | Reduces typing. | LOW | |

Edge cases the backend must handle:
- No short selling: selling more than held is rejected, not allowed to go negative.
- Selling the full position: floating point makes "sell 10.4999999 of 10.5" leave dust. Enforce a max of 4 decimals on quantity input and treat a remaining quantity below an epsilon (e.g. 1e-6) as zero, deleting the row. Optionally clamp a sell within epsilon of the held quantity to the full held quantity.
- Price is read from the server cache at execution time, not from what the user saw. That is fine for simulated fills; show the actual fill price in the confirmation.
- No price available for the ticker: reject with "No live price for XYZ. Add it to your watchlist first."
- The trade bar should allow trading only tickers the system is pricing (watchlist or held). If the user types a ticker not in the watchlist, recommend rejecting with a hint rather than silently adding it (see Spec Gaps).
- Atomicity: one DB transaction (cash, position upsert/delete, trade row) plus a lock so a manual trade and an LLM auto-trade cannot interleave and double-spend cash.
- Do not allow cash to go negative through rounding: compare with a small epsilon. Round displayed money to cents; consider rounding the fill price to 4 decimals at execution to keep numbers tidy.

#### 8. AI chat panel (docked, collapsible, inline confirmations)

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Message list, text input, Enter to send (Shift+Enter newline) | Basic chat UX. | LOW | Auto-scroll to the newest message. |
| Optimistic user message plus loading indicator | Spec: no streaming, just a loading indicator. | LOW | Typing-dots bubble. Disable input (or block double-send) while pending. |
| Inline action confirmations under the assistant message | Spec. Core of the "agentic" demo. | MEDIUM | See display rules below. |
| Error handling that does not break the chat | LLM down, bad key, timeout, or malformed JSON must not freeze the UI. | MEDIUM | Show an assistant-style error bubble ("I could not reach the AI service. Check OPENROUTER_API_KEY.") and re-enable input. |
| History survives a page reload | `chat_messages` is persisted, so a reload that shows an empty chat looks buggy. | LOW-MEDIUM | Needs a way to read history (see Spec Gaps). Re-render persisted `actions` as the same confirmation chips. |
| Collapse and expand | Spec. | LOW | Layout must reflow so the freed space goes to the main panels. |
| UI refreshes after the AI trades | If the chat says "Bought" and the positions table does not change, trust is lost. | LOW | Chat success triggers the same refetch as a manual trade (shared store action). |

Action confirmation display rules (recommended):
- Render from the structured `actions` data the API returns, NOT by parsing the LLM prose. The API response should carry a per-action result, e.g. `{type: "trade", ticker, side, quantity, status: "executed" | "failed", price, total, error}` and `{type: "watchlist", ticker, action, status, error}`.
- Executed trade chip (green): "Bought 10 AAPL @ $190.12 (total $1,901.20)". Sell chip likewise, optionally with realized P&L.
- Failed trade chip (red/amber): "Could not buy 1,000 TSLA: insufficient cash (have $8,098.80)". Shown even if the LLM prose says otherwise.
- Watchlist chip: "Added PYPL to watchlist" or "Removed PYPL".
- Order of execution within one response: apply `watchlist_changes` (adds) before `trades`, so "add PYPL and buy 5" can succeed in one turn. Process trades in array order; each is validated against the cash and positions as updated by the previous trade.
- The LLM text can contradict outcomes ("I bought 10 AAPL" while the trade failed). The chips are the source of truth. Mitigate in the system prompt (phrase intent as "placing" and let the chips confirm) and optionally in the UI by visually emphasizing chips.

#### 9. Cross-cutting UX

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Consistent number formatting | Terminal credibility. | LOW | `$1,234.56`, signed P&L (`+`/`-`), 2-decimal prices, tabular numerals. One shared formatter module. |
| Dark theme per spec, with spec accent colors | Spec. | LOW | `#0d1117`-ish background, yellow `#ecad0a` accent, blue `#209dd7`, purple `#753991` for submit buttons. Green/red for P&L must remain legible on dark and be paired with +/- signs. |
| Initial loading state without layout shift | First impression. | LOW | Fixed grid layout; panels render skeleton or "--". |
| Stale-data signal when disconnected | Traders do not trust frozen prices. | LOW | Dim prices or show a banner when the dot is red. See differentiators. |
| Responsive-ish layout | Spec: desktop-first, functional on tablet. | MEDIUM | CSS grid with a fixed desktop layout; stack or scroll on narrow screens. No mobile redesign. |

---

### Differentiators (Competitive Advantage)

These are where the demo gets its "wow". Pick a few; the core value is "live prices plus instant trades plus an AI that acts". Items are ordered by value per effort.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| AI executes trades and manages the watchlist in plain language, with inline result chips | The headline differentiator of the whole project (already in spec). Most paper-trading apps have no copilot. | MEDIUM | Quality lives in the system prompt, per-action results, and refreshing all panels afterward. |
| Portfolio-aware AI analysis (concentration, P&L, cash %, "what is my biggest risk?") | Makes the AI useful beyond a trade-entry shortcut. | LOW | Mostly free: portfolio context is already injected. Include computed fields (weights, cash %, unrealized P&L %) in the context so the model does not do arithmetic. |
| Starter prompt chips in an empty chat ("Analyze my portfolio", "Buy 5 NVDA", "Add PYPL to my watchlist") | Removes blank-page friction and guides demo viewers to the best flows. | LOW | Click inserts or sends the text. |
| Visual drama from the simulator (flashes, 2-5% shock events visible in heatmap and chart) | Already built into market data. The UI just needs to not hide it. | LOW | Make sure flashes and the heatmap respond visibly to big moves. |
| Average-cost line on the main chart when the selected ticker is held | Instantly shows whether you are above or below your entry. | LOW | One `createPriceLine` call. Strong "this is a real workstation" signal for the cost. |
| Click a positions-table or heatmap cell to select that ticker in the main chart | Cross-panel linking feels like a terminal. | LOW | Shared `selectedTicker` store. |
| Candlesticks built client-side from ticks (e.g. 5s buckets) with a line/candle toggle | Looks professional; cheap with Lightweight Charts. | MEDIUM | Needs OHLC bucketing logic on the client; start with line and add if time allows. |
| Quick-quantity helpers in the trade bar (25/50/100% of position for sells, "max" for buys) and an estimated cost/proceeds readout | Speeds up demo flows and reduces rejection errors. | LOW | Uses the live price and cash from the stores. |
| Total P&L and day P&L in the header with color | Gives the portfolio a "scoreboard". | LOW | Day P&L needs per-position baseline; total P&L is trivial. Prefer total only. |
| Collapsed-chat unread badge | Tells the user the AI responded or traded while hidden. | LOW | Only matters if the AI can act while collapsed (it cannot, since requests are user-initiated); skip unless trivial. |
| Dimmed prices plus "Reconnecting..." banner when the stream drops | Honest degraded state; tests well in the SSE-resilience e2e. | LOW | Driven by the connection status store. |
| Keyboard shortcut to focus chat or trade bar | Terminal feel. | LOW | Nice-to-have, P3. |

---

### Anti-Features (Commonly Requested, Often Problematic)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Trade confirmation dialogs (manual or AI) | "Safety". | Explicit spec decision: fake money, fluid agentic demo. Adds clicks and breaks E2E timing. | Inline result feedback after the fact (trade bar line, chat chips). |
| Limit, stop, and stop-limit orders; partial fills; fees; order book | Real brokerages have them. | Spec: market orders only. Each one multiplies portfolio math and test surface. | Market orders at the current cache price. |
| Short selling and margin | Common trader ask. | Negative positions break treemap weights, avg-cost math, and the "sell more than owned" rule. | Reject sells that exceed holdings. |
| Dollar-amount orders ("buy $500 of AAPL") | Popular in Robinhood. | New input mode and rounding rules, not in spec. | Fractional share quantities (already supported). The LLM can convert dollars to shares itself if asked. |
| Token-by-token chat streaming | Feels modern. | Spec: complete JSON response; structured output plus auto-execution needs the whole response first. | Typing-dots loading indicator. |
| Chat tool-call trace or "thinking" display | Looks impressive. | Not supported by the single-shot structured-output design; adds noise. | Action chips only. |
| Historical price backfill and timeframe selectors (1D, 1W, 1M) | Standard on chart products. | PROJECT.md out of scope; the simulator has no history; Massive history costs API calls. | Charts accumulate from page load; label this honestly ("since page load"). |
| Technical indicators, drawing tools, multi-chart layouts | "Pro" feel. | Large surface, no value for a simulated random walk. | One clean chart. |
| Sparklines persisted across reloads | Nicer UX. | Needs server-side tick storage; spec says client-accumulated since page load. | Let them refill. |
| Re-sorting watchlist by change % | Looks dynamic. | Rows jumping under the cursor while the user tries to click or trade. | Stable order. |
| Real news, fundamentals, analyst ratings, earnings calendar | Terminal flavor. | Extra data sources, none in spec; the LLM has no news access either, so it would hallucinate. | Out of scope. State that the AI only knows portfolio, watchlist, and prices. |
| Multi-conversation chat, chat search, voice, file upload | Chat products have them. | Single user, single thread, persistent `chat_messages` only. | One thread. |
| "Undo trade" via AI or UI | Looks forgiving. | Trades are an append-only log; undo means fake reversal rows and price confusion. | Make the user trade back out manually. |
| Price alerts and notifications | Typical trader feature. | Needs persistence, a scheduler, and a delivery channel. | Out of scope. |
| Light theme toggle, theme editor | Nice for users. | Dark terminal aesthetic is the identity. | Dark only. |
| Draggable and resizable panel grid | Terminal workspaces are customizable. | High complexity, fragile with charts and resize observers, no demo value. | Fixed CSS grid layout. |
| Live tick appended to the P&L chart every 500ms | Looks lively. | Chart data would no longer match persisted snapshots; header already shows live value. | Refresh from snapshots (30s + after trades). |
| Auth, accounts, leaderboard | Gamification. | Single-user by design. | `user_id="default"` keeps it possible later. |
| In-app "reset portfolio" button | Helpful after a bad run. | Not in spec; needs destructive-action UX and API. | Document the stop script plus removing the Docker volume (`finally-data`). Candidate for a later milestone. |

---

## Spec Gaps and Ambiguities (resolve in requirements)

These are the places where PLAN.md is silent or ambiguous and an implementer would otherwise have to guess.

| # | Gap | Impact | Recommended resolution |
|---|-----|--------|------------------------|
| 1 | No endpoint to READ chat history (`/api/chat` is POST only) | Reload loses the visible conversation even though it is persisted. | Add `GET /api/chat` (recent N messages with `actions`). Small, needed for a polished panel. |
| 2 | Daily-change baseline is not carried by SSE (`previous_price` is tick-over-tick) | Watchlist "daily change %" has no data source. | `GET /api/watchlist` returns an open/prev-close price per ticker; client computes the % against live prices. |
| 3 | No ticker-existence validation exists (simulator accepts anything) | Hallucinated or typo tickers get fake prices; Massive tickers may never receive prices. | Format validation always; existence check for Massive; for simulator accept any well-formed symbol or use a small allowlist. Decide explicitly. |
| 4 | Removing a watchlist ticker removes its cache price, but positions may still need it | Open position becomes unpriceable: total value wrong, heatmap and sells break. | Track `watchlist UNION positions` in the market data source. Removal from the watchlist must not stop pricing a held ticker (or block removal with a message). |
| 5 | Can the trade bar or the LLM trade a ticker that is not on the watchlist? | Unspecified. The price cache only holds tracked tickers. | Require a live price. Reject with a hint, or (AI flow) allow `watchlist_changes` add before trades in the same response. |
| 6 | "Error is included in the chat response so the LLM can inform the user" | Ambiguous: second LLM call, or just include errors in the response? | Recommend no second LLM call. Return per-action results alongside the message and render chips. Prompt the LLM not to claim success prematurely. |
| 7 | Mock-mode behavior is "deterministic" but unspecified | E2E tests need a stable contract. | Define it: e.g. message contains "buy" gives a fixed buy of a fixed ticker/quantity; contains "add" gives a watchlist add; otherwise a fixed analysis message. Document in the requirements. |
| 8 | No endpoint for trade history (`trades` table is written, never read) | No trade log view. | Acceptable to omit in v1 (chips and positions are enough). Do not add a history page now. |
| 9 | "% change" column in positions table undefined | Two plausible meanings (daily versus versus cost). | Use unrealized P&L % versus average cost. |
| 10 | Startup snapshot is not specified | P&L chart empty on a fresh start. | Write an initial snapshot at DB seed or startup. |
| 11 | Quantity precision and minimum are unspecified | Float dust, unreadable quantities. | Max 4 decimals; minimum 0.0001; zero-epsilon 1e-6 when closing a position. |
| 12 | Behavior when SSE is quiet (Massive polls every 15s and the stream only emits on change) | A "stale" heuristic based on time since last message would flicker red. | Drive the status dot from `EventSource` state only, or make any stale threshold larger than the poll interval. |

---

## P&L and Position Math (reference for requirements and tests)

Use average-cost accounting, which the schema (`avg_cost` per position) already implies.

- Buy `q` at price `p` on an existing position (`Q`, `A`): `new_qty = Q + q`; `new_avg_cost = (Q*A + q*p) / (Q + q)`. `cash -= q*p`.
- Sell `q` at price `p`: require `q <= Q` (within epsilon). `new_qty = Q - q`; `avg_cost` is UNCHANGED on a partial sell. This is the standard average-cost rule and the classic bug (some implementations wrongly recompute it). `cash += q*p`. Realized P&L for this sell is `(p - A) * q` (not stored; display in the chip only if wanted).
- Full sell: when `new_qty <= 1e-6`, delete the position row. A later re-buy starts with a fresh `avg_cost`.
- Unrealized P&L per position: `(price - avg_cost) * qty`. Unrealized %: `price / avg_cost - 1`.
- Market value: `qty * price`. Total portfolio value: `cash + sum(market_value)`. Total return: `total_value - 10000`.
- Selling at a loss: `avg_cost` stays, price below it gives negative unrealized P&L before the sale. The spec test list includes this; ensure realized loss math is covered.
- Heatmap weight: `market_value / sum(market_value)` over positions (cash excluded).
- Fractional shares: store as REAL; accept up to 4 decimals; buy cost is `q*p` rounded to cents for cash accounting (decide once and test it).
- Worked example for tests: buy 10 @ 100 (avg 100), buy 10 @ 120 (avg 110, qty 20), sell 5 @ 130 (qty 15, avg still 110, realized +100), price 125 gives unrealized `(125-110)*15 = +225`, +13.6%.

---

## Feature Dependencies

```
Backend: DB + seed + startup snapshot
    ├──requires──> Portfolio API (GET, POST /trade, history)
    │                  ├──requires──> Price cache + market data (DONE)
    │                  ├──enables──> Trade bar
    │                  ├──enables──> Positions table
    │                  ├──enables──> Heatmap
    │                  ├──enables──> P&L chart (needs snapshots)
    │                  └──enables──> Header (cash / total value)
    ├──requires──> Watchlist API (+ open/prev-close baseline field)
    │                  └──enables──> Watchlist panel
    └──requires──> Chat API (+ chat history read endpoint)
                       ├──requires──> Portfolio API (trade execution path shared with manual trades)
                       ├──requires──> Watchlist API (watchlist_changes)
                       ├──requires──> LLM_MOCK contract (E2E)
                       └──enables──> Chat panel + inline action chips

Frontend foundation: SSE client + connection state + shared stores
    ├──enables──> Header status dot, live total value
    ├──enables──> Watchlist flash + per-ticker price buffer
    │                  ├──enables──> Sparklines
    │                  └──enables──> Main chart (selected ticker)
    └──enables──> Live positions / heatmap valuation

Selected-ticker store ──links──> Watchlist, Main chart, Trade bar prefill, Positions row click, Heatmap click
Post-trade refetch action (shared) ──used by──> Trade bar AND Chat panel (after AI actions)

Held-ticker pricing rule (watchlist UNION positions) ──protects──> Positions, Heatmap, Header value

Ticker validator (shared) ──used by──> Watchlist add, Trade bar, LLM watchlist_changes, LLM trades
```

### Dependency Notes

- **Heatmap, positions table, and header value all depend on the live price map, not on the watchlist UI.** Build the SSE store and price buffers first; panels are consumers.
- **Chat requires the trade and watchlist execution paths to be reusable service functions**, not logic embedded in route handlers, so the LLM flow and the manual flow share validation and the lock.
- **Sparklines and the main chart share one buffer.** Splitting them into per-component state forces a rewrite.
- **P&L chart depends on snapshot writes** (startup, every 30s, post-trade). Without the startup snapshot it is empty on first view.
- **Watchlist removal conflicts with held positions** unless the pricing set is the union of watchlist and holdings (gap 4).
- **Playwright "AI chat" scenario depends on a defined mock contract** (gap 7).

---

## MVP Definition

Everything in PLAN.md is in scope for v1 (PROJECT.md: build the full spec). The prioritization below orders work inside that scope and shows what to cut first if time runs out.

### Launch With (v1)

- [ ] Backend portfolio/trade service with average-cost math, validation, atomic transactions, snapshot writes (startup, 30s, post-trade). It is the core of the product.
- [ ] Watchlist API with open/prev-close baseline, format validation, duplicate handling, held-ticker pricing rule
- [ ] SSE client with connection state, shared price store, per-ticker ring buffers
- [ ] Header (live total value, cash, total P&L, status dot)
- [ ] Watchlist panel (flash, daily change %, sparkline, add/remove, select)
- [ ] Main chart (line/area, live, seeded from the buffer, timestamp dedupe)
- [ ] Trade bar with inline feedback and validation messages
- [ ] Positions table (live P&L, empty state, fractional display)
- [ ] Heatmap (diverging color, labels, empty state, throttled)
- [ ] P&L chart (snapshots, $10k reference line, empty state)
- [ ] Chat panel + `POST /api/chat` + history read endpoint + action chips + error handling + `LLM_MOCK` contract
- [ ] Per-feature empty, loading, and error states (they are the e2e "fresh start" scenario)

### Add After Validation (v1.x)

- [ ] Average-cost price line on the main chart. Trigger: main chart is stable.
- [ ] Cross-panel selection (positions/heatmap click selects ticker). Trigger: selected-ticker store exists.
- [ ] Quick-quantity buttons and estimated cost readout. Trigger: trade bar done.
- [ ] Starter prompt chips in chat. Trigger: chat done.
- [ ] Dimmed prices and a banner when disconnected. Trigger: status dot done.

### Future Consideration (v2+)

- [ ] Candlestick mode with client-side OHLC bucketing
- [ ] Reset-portfolio button
- [ ] Trade history view (needs `GET /api/trades`)
- [ ] Persisted sparkline or price history (needs server-side tick storage)
- [ ] Symbol search or allowlist with company names

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Trade execution + average-cost math + validation | HIGH | MEDIUM | P1 |
| SSE client + connection status + price store | HIGH | MEDIUM | P1 |
| Watchlist panel (flash, change %, sparkline) | HIGH | MEDIUM | P1 |
| Header (live value, cash, status) | HIGH | LOW | P1 |
| Trade bar with feedback | HIGH | LOW | P1 |
| Positions table | HIGH | LOW | P1 |
| AI chat with auto-executed trades + chips | HIGH | MEDIUM | P1 |
| Main chart (live line) | HIGH | MEDIUM | P1 |
| Heatmap (treemap) | MEDIUM-HIGH | MEDIUM | P1 (spec; visual centerpiece) |
| P&L chart | MEDIUM-HIGH | MEDIUM | P1 (spec; e2e requires it) |
| Ticker validation + duplicate handling | MEDIUM | LOW | P1 |
| Chat history read endpoint | MEDIUM | LOW | P1 |
| Empty / loading / error states everywhere | HIGH | LOW-MEDIUM | P1 |
| Total P&L in header | MEDIUM | LOW | P2 |
| Avg-cost line on chart | MEDIUM | LOW | P2 |
| Cross-panel selection | MEDIUM | LOW | P2 |
| Quick-quantity helpers | MEDIUM | LOW | P2 |
| Starter prompt chips | MEDIUM | LOW | P2 |
| Disconnected-state dimming | LOW-MEDIUM | LOW | P2 |
| Candlestick mode | LOW-MEDIUM | MEDIUM | P3 |
| Keyboard shortcuts | LOW | LOW | P3 |

**Priority key:**
- P1: Must have for launch (inside the PLAN.md scope or required to make a PLAN.md feature work properly)
- P2: Should have, cheap polish that amplifies the demo
- P3: Nice to have, defer

---

## Competitor Feature Analysis

| Feature | Robinhood / Webull | thinkorswim paperMoney / TradingView paper trading | Finviz map | Our Approach |
|---------|--------------------|----------------------------------------------------|------------|--------------|
| Watchlist | Price, day change, sparkline; color flashes | Dense table, many columns, drag to reorder | n/a | Ticker, price, day %, sparkline; fixed order; flash per spec |
| Chart | Line chart with timeframes | Full charting with indicators | n/a | Single live line chart from page load; no timeframes |
| Order entry | Confirmation screen, fractional shares and dollar orders | Full order ticket (limit/stop/etc.) | n/a | One-line market-order bar, no confirmation, fractional shares |
| Positions | Average cost, total return, today's return | Positions tab with P&L open/day | n/a | Per-spec columns; average-cost P&L |
| Portfolio visual | Line chart of equity | P&L charts and risk profile | Treemap sized by market cap, colored by % change | Treemap sized by position weight, colored by unrealized P&L % |
| AI copilot | Basic assistants (Q&A, summaries); no auto-trading | None | n/a | Chat that analyzes and executes trades and watchlist changes with inline results (differentiator) |
| Connection state | Hidden | Subtle status indicators | n/a | Explicit dot (green/yellow/red) |

## Sources

- `planning/PLAN.md`, `.planning/PROJECT.md` (scope; authoritative)
- `planning/MARKET_DATA_SUMMARY.md`, `backend/CLAUDE.md`, `backend/app/market/simulator.py`, `backend/app/market/stream.py` (verified codebase facts; HIGH)
- Lightweight Charts maintainer docs via Context7 (`/tradingview/lightweight-charts`): strictly ascending unique times, `update()` semantics, client-only rendering under Next.js SSR (HIGH)
- General knowledge of Robinhood, Webull, thinkorswim paperMoney, TradingView paper trading, Finviz map, Investopedia simulator, Bloomberg conventions (MEDIUM; not re-verified against live product pages)
- Average-cost accounting convention (standard brokerage practice; MEDIUM-HIGH)

---
*Feature research for: AI-assisted simulated trading workstation*
*Researched: 2026-10-27*
