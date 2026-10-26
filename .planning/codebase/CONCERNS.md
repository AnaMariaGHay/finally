---
last_mapped_commit: b97d1fef40a1301a65818a1554fd2d67cc274d27
last_mapped_at: 2026-10-26
---
# Codebase Concerns

**Analysis Date:** 2026-10-26

## Critical Missing Infrastructure

### FastAPI Application & API Routes

**Issue:** No main FastAPI application or API endpoints implemented.

**Files affected:** 
- `backend/app/` — no `main.py` or entry point
- No routers for `/api/portfolio`, `/api/watchlist`, `/api/chat`, `/api/health`

**Impact:** 
- Cannot run the backend at all — no server to start
- Cannot test API contracts with frontend
- Blocks all downstream work

**Fix approach:** 
Create `backend/app/main.py` with FastAPI app initialization, routers for portfolio/watchlist/chat/health endpoints, startup/shutdown lifecycle management. Wire up PriceCache and market data source.

---

### Database Layer

**Issue:** No SQLite schema, migrations, or ORM/query layer implemented.

**Files affected:**
- `backend/db/` — no schema definitions
- No database initialization logic
- No models or queries for users, positions, trades, chat history

**Impact:**
- Cannot persist any state (positions, portfolio snapshots, chat history)
- Can only simulate; cannot track actual trades
- Portfolio P&L calculations impossible without trade history
- LLM chat integration cannot save conversation history

**Fix approach:**
Create `backend/db/schema.sql` with table definitions for `users_profile`, `positions`, `trades`, `portfolio_snapshots`, `watchlist`, `chat_messages`. Implement lazy initialization on first request. Use a simple SQLite query layer (no ORM required for simplicity).

---

### Frontend Implementation

**Issue:** Next.js frontend project does not exist.

**Files affected:**
- `frontend/` — directory does not exist
- No pages, components, or static assets
- No TypeScript/React code

**Impact:**
- Cannot display UI to users
- Cannot send orders, view charts, or chat with AI
- Project is backend-only, non-functional as a product

**Fix approach:**
Create full Next.js TypeScript project with SSE integration, chart components, portfolio visualization, and AI chat interface. Export as static files for serving from FastAPI.

---

### Docker & Deployment

**Issue:** No Dockerfile, docker-compose, or start/stop scripts.

**Files affected:**
- `Dockerfile` — missing
- `docker-compose.yml` — missing
- `scripts/start_mac.sh`, `scripts/stop_mac.sh`, `scripts/start_windows.ps1`, `scripts/stop_windows.ps1` — all missing

**Impact:**
- Cannot package or deploy the application
- Users cannot run `docker run` or start scripts
- No production-ready delivery method

**Fix approach:**
Create multi-stage Dockerfile (Node → Python), docker-compose.yml for development, and platform-specific start/stop scripts that handle volume mounts and env-file loading.

---

## Market Data Source Concerns

### Cholesky Decomposition Failure

**What happens:** In `backend/app/market/simulator.py` line 172, the correlation matrix is decomposed via `np.linalg.cholesky()`. If the matrix is not positive-definite (due to numerical precision issues or invalid correlation values), this will raise `numpy.linalg.LinAlgError`.

**Why it's wrong:**
- No error handling around the Cholesky call
- If it fails, the simulator crashes
- Correlation values are hardcoded, but future code might compute them dynamically
- Rebuilding Cholesky on every add/remove ticker could compound numerical errors

**Do this instead:**
Wrap `_rebuild_cholesky()` in a try-except. If decomposition fails, fall back to identity matrix (no correlation). Log a warning. Add epsilon to diagonal of correlation matrix for numerical stability:

```python
def _rebuild_cholesky(self) -> None:
    n = len(self._tickers)
    if n <= 1:
        self._cholesky = None
        return
    
    corr = np.eye(n)
    for i in range(n):
        for j in range(i + 1, n):
            rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
            corr[i, j] = rho
            corr[j, i] = rho
    
    # Add epsilon to diagonal for numerical stability
    corr += 1e-8 * np.eye(n)
    
    try:
        self._cholesky = np.linalg.cholesky(corr)
    except np.linalg.LinAlgError:
        logger.warning("Cholesky decomposition failed; using identity (no correlation)")
        self._cholesky = None
```

---

### Massive API Timeout Handling

**What happens:** In `backend/app/market/massive_client.py` line 97, `asyncio.to_thread(self._fetch_snapshots)` runs the synchronous REST client in a thread. The REST call has no timeout set.

**Why it's wrong:**
- If the Massive API is slow or unresponsive, the thread will block indefinitely
- No exponential backoff or intelligent retry
- A single slow API call can cascade and block the entire event loop

**Do this instead:**
Set a timeout on the REST client or wrap the `to_thread` call with `asyncio.timeout()`:

```python
async def _poll_once(self) -> None:
    if not self._tickers or not self._client:
        return
    
    try:
        snapshots = await asyncio.timeout(5.0)(
            asyncio.to_thread(self._fetch_snapshots)
        )
        # ... rest of logic
    except asyncio.TimeoutError:
        logger.warning("Massive API poll timed out after 5s; skipping cycle")
    except Exception as e:
        logger.error("Massive poll failed: %s", e)
```

---

### GBM Price Unbounded Growth

**What happens:** The GBM model in `backend/app/market/simulator.py` uses `S(t+dt) = S(t) * exp(drift + diffusion)`. Prices grow at the drift rate (e.g., 5% annually). Over years, prices could reach unrealistic values (e.g., $1,000,000 for a single share).

**Why it's wrong:**
- While mathematically correct GBM behavior, it's unrealistic for a trading simulator
- Portfolio visualizations and calculations could suffer floating-point precision issues
- Visually confusing to users who expect price ranges to stay reasonable

**Do this instead:**
Consider adding a maximum price cap per ticker (e.g., 3x the seed price) and reset if exceeded. Or use mean-reversion model instead of pure GBM. Or accept unbounded growth and handle floating-point carefully in visualizations (e.g., scientific notation for very large values).

---

## SSE Streaming Concerns

### SSE Generator Resource Cleanup

**What happens:** In `backend/app/market/stream.py`, the `_generate_events()` async generator yields events in an infinite loop (line 69). If the client disconnects, the `await request.is_disconnected()` check (line 71) eventually detects it and breaks. However, the check happens only after `asyncio.sleep(0.5)`.

**Why it's wrong:**
- If a client connects and immediately disconnects, the generator still runs for 500ms before detecting it
- On a busy server with hundreds of concurrent clients, stale generators could accumulate and waste CPU
- No explicit cleanup or resource release

**Do this instead:**
Use a shorter sleep interval for disconnect detection, or add a task that explicitly cancels generators on client disconnect. Consider tracking active connections and logging them for monitoring:

```python

# Track active connections

_active_connections = set()

async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    conn_id = id(request)
    _active_connections.add(conn_id)
    logger.debug(f"Active connections: {len(_active_connections)}")
    
    try:
        yield "retry: 1000\n\n"
        # ... rest of logic
    finally:
        _active_connections.discard(conn_id)
```

---

### SSE Payload Format Mismatch

**What happens:** In `backend/app/market/stream.py` line 81, the SSE event data is formatted as:

```
data: {"AAPL": {...}, "GOOGL": {...}}
```

But the docstring (line 33) doesn't clarify this format. Frontend might expect a different structure (e.g., array instead of object keyed by ticker).

**Why it's wrong:**
- Frontend implementation must exactly match this format or parsing will fail
- No schema documentation or validation on either side

**Do this instead:**
Define and document a strict schema. Consider adding a `/api/schema` endpoint that returns the expected SSE format. Or use a versioned format with explicit schema negotiation.

---

## Version Counter Overflow

**What happens:** In `backend/app/market/cache.py` line 21, `self._version` is a plain `int`. Every price update increments it (line 41). The SSE stream uses this version for change detection (line 75 in stream.py).

**Why it's wrong:**
- Python integers don't overflow, but after years of operation at ~2 updates/sec, _version could become very large
- Comparison logic assumes version values stay reasonable
- No reset or wraparound mechanism

**Do this instead:**
Use a bounded version counter with modular arithmetic, or simply accept that Python integers are arbitrary precision and the code is fine as-is. If concerned, add a comment explaining this is not a risk in Python.

---

## Input Validation & Security

### Ticker Symbol Validation Missing

**What happens:** Ticker symbols are passed through the API with no validation. In `backend/app/market/simulator.py` line 69 and `factory.py`, tickers are used directly as dictionary keys.

**Why it's wrong:**
- User-submitted tickers could contain special characters, SQL injection payloads (if later used in SQL without parameterization), or extremely long strings
- `SimulatorDataSource.add_ticker()` blindly appends to `_tickers` without sanitization
- Massive API client uppercases and strips in `massive_client.py` line 67, but this is inconsistent with simulator

**Do this instead:**
Validate all tickers at API boundary. Create a utility function:

```python
def validate_ticker(ticker: str) -> str:
    """Validate and normalize a ticker symbol."""
    ticker = ticker.upper().strip()
    if not ticker:
        raise ValueError("Ticker cannot be empty")
    if not ticker.isalnum():
        raise ValueError("Ticker must be alphanumeric")
    if len(ticker) > 10:
        raise ValueError("Ticker must be <= 10 chars")
    return ticker
```

Use this in all entry points: API routes, market data source methods, and database queries.

---

## Testing Gaps

### SSE Streaming Not Tested

**Issue:** No tests for `backend/app/market/stream.py`.

**Files affected:**
- `backend/tests/` — no `test_stream.py`

**Why it's wrong:**
- The SSE endpoint is critical for frontend data delivery
- No verification that events are properly formatted, retry directive works, or disconnect handling works
- No test for version-based change detection

**Priority:** High

**Test coverage goal:**
- SSE format compliance
- Disconnect detection
- Event payload correctness
- Version counter behavior

---

### No Integration Tests for Full API

**Issue:** Tests only cover market data module. No tests for portfolio logic, trade execution, or chat flow.

**Files affected:**
- `backend/tests/` — missing `test_api.py`, `test_portfolio.py`, `test_database.py`

**Why it's wrong:**
- Cannot verify end-to-end request/response contracts
- Trade execution logic (buy/sell validation) untested
- Portfolio calculations (P&L, unrealized gains) untested
- Chat message parsing and trade auto-execution untested

**Priority:** High (blocks any API work)

---

### No E2E Tests

**Issue:** `test/` directory exists but is empty. No Playwright tests or docker-compose.test.yml.

**Why it's wrong:**
- Cannot verify full user workflows (connect → watch prices → buy → chat → view portfolio)
- Cannot test UI animations, SSE reconnection, or chart rendering
- Deployment risk is very high without E2E validation

**Priority:** High

---

## Production Readiness

### No Health Check Endpoint

**Issue:** `/api/health` endpoint not implemented.

**Impact:**
- Docker health checks cannot verify server is responsive
- Kubernetes/cloud platforms cannot auto-restart failed containers
- Load balancers cannot detect dead backends

**Fix approach:**
Add simple health check that verifies:
- Server is running
- Database is accessible (when database layer exists)
- Market data source is streaming (optional, but useful)

```python
@app.get("/api/health")
async def health_check():
    return {
        "status": "ok",
        "timestamp": time.time(),
        "services": {
            "market_data": "ok" or "error"
        }
    }
```

---

### No Graceful Shutdown

**Issue:** No shutdown handlers in main FastAPI app. Market data source tasks may not stop cleanly.

**Impact:**
- Simulator background task left running
- Database connections not flushed
- Portfolio snapshots might not be recorded
- Massive API poller may not cancel properly

**Fix approach:**
Use FastAPI `@app.on_event("shutdown")` to await `source.stop()` and database cleanup.

---

### Logging Not Configured

**Issue:** Code uses `logging.getLogger(__name__)` (e.g., `simulator.py` line 25) but there's no root logger configuration.

**Impact:**
- Log messages may not appear or may go to stderr without formatting
- No way to control log level from environment
- Hard to debug in production without log visibility

**Fix approach:**
Configure logging in main app startup:

```python
import logging

logging.basicConfig(
    level=os.getenv("LOG_LEVEL", "INFO"),
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)
```

---

## Race Conditions

### SimulatorDataSource.stop() Race Condition

**What happens:** In `backend/app/market/simulator.py` line 232-239, `stop()` accesses `self._task` without a lock. If two calls happen concurrently, they could both check `if self._task and not self._task.done()` and both attempt to cancel.

**Why it's wrong:**
- While Python's GIL prevents true race conditions on simple operations, explicit synchronization is safer
- Concurrent calls to `stop()` could leave inconsistent state
- No indication that `stop()` is idempotent or safe to call multiple times

**Do this instead:**
Add a lock or flag:

```python
async def stop(self) -> None:
    if self._stopping:
        return
    self._stopping = True
    
    if self._task and not self._task.done():
        self._task.cancel()
        try:
            await self._task
        except asyncio.CancelledError:
            pass
    self._task = None
    logger.info("Simulator stopped")
```

---

## Scaling & Performance

### No Rate Limiting on API Endpoints

**Issue:** When API endpoints are implemented, they will have no rate limiting.

**Impact:**
- Users could hammer the API with unlimited requests
- Massive API quota could be exhausted by a single misbehaving client
- No protection against DDoS or runaway scripts

**Fix approach:**
Use `slowapi` or similar library to add rate limiting:

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/api/portfolio/trade")
@limiter.limit("10/minute")
async def execute_trade(request: Request, ...):
    ...
```

---

### Massive API Cost

**Issue:** The Massive API charges per call. Current implementation polls all tickers on every interval (default 15s).

**Impact:**
- 10 tickers × 4 calls/minute = 40 calls/minute = 57,600 calls/day
- Free tier: 5 calls/minute max, so will hit rate limits immediately
- Paid tiers could become expensive at scale

**Fix approach:**
- Document the cost clearly in the plan
- Implement batch updates or subscription-based polling if available
- Cache responses longer or use simulator by default in documentation
- Alert users if MASSIVE_API_KEY is set (they're incurring charges)

---

## Data Integrity Concerns

### No Trade Validation

**Issue:** When trade execution is implemented, there's no mechanism described for:
- Verifying sufficient cash for buy orders
- Verifying sufficient shares for sell orders
- Handling partial fills or rejected trades
- Rolling back trades on failure

**Impact:**
- Portfolio could enter invalid state (negative cash, negative shares)
- P&L calculations wrong if trades weren't actually executed
- User-facing chat interface might show "trade executed" when it failed

**Fix approach:**
Implement strict validation:

```python
async def execute_trade(ticker: str, side: str, quantity: float) -> TradeResult:
    if side == "buy":
        cost = current_price * quantity
        if user.cash_balance < cost:
            return TradeResult(success=False, error="Insufficient cash")
    elif side == "sell":
        position = get_position(user, ticker)
        if not position or position.quantity < quantity:
            return TradeResult(success=False, error="Insufficient shares")
    
    # Execute atomically
    position.update(...)
    user.cash_balance -= cost
    record_trade(...)
    record_portfolio_snapshot(...)
    
    return TradeResult(success=True, ...)
```

---

## Missing Components Blocking Development

### LLM Integration Not Started

**Issue:** No implementation for the chat endpoint or LLM client.

**Impact:**
- Cannot execute trades via chat
- Cannot save conversation history
- AI copilot features blocked

**Requires:**
- `backend/app/llm/` module with LiteLLM client
- Structured output parsing for trade/watchlist actions
- Database layer for storing chat history
- Integration with portfolio service

---

### Portfolio Service Not Started

**Issue:** No implementation for portfolio calculations, P&L tracking, or trade history.

**Impact:**
- Cannot calculate unrealized/realized gains
- Cannot display position-level P&L
- Cannot show portfolio snapshots over time
- Trade execution logic doesn't exist

**Requires:**
- `backend/app/portfolio/` module
- Database queries for positions and trades
- P&L calculation logic with cost basis tracking
- Portfolio snapshot recording (every 30s + after each trade)

---

### Watchlist Management Not Started

**Issue:** No watchlist CRUD logic implemented.

**Impact:**
- Cannot add/remove tickers from watchlist
- Cannot display watchlist in SSE stream
- Cannot sync watchlist across market data source and database

**Requires:**
- `backend/app/watchlist/` module or integrated into portfolio
- Database queries for watchlist
- Synchronization between API requests and market data source

---

## Documentation Gaps

### API Contract Not Defined

**Issue:** No OpenAPI/Swagger documentation or request/response schemas defined.

**Impact:**
- Frontend implementation must guess expected formats
- No way to auto-generate SDK or client code
- Integration testing difficult without known contract

**Fix approach:**
Define Pydantic models for all request/response bodies. Use FastAPI's automatic OpenAPI generation. Document all error responses.

---

## Security Review Needed

### No Authentication/Authorization

**Issue:** The plan notes this is single-user, but there's no mechanism to prevent a user from accessing another user's data (if multi-user is ever added). The `user_id="default"` hardcoding is a placeholder.

**Impact:**
- No audit trail of who did what
- No multi-user support without code changes
- Data isolation not enforced

**Fix approach:** (If multi-user is ever needed)
- Add user_id extraction from JWT or session
- Implement middleware to inject user_id into all queries
- Add database constraints to prevent cross-user data leakage
- Audit log all trade executions

---

## Summary Table

| Area | Severity | Impact | Blocking |
|------|----------|--------|----------|
| Missing FastAPI app | **Critical** | Cannot start server | Yes |
| Missing database layer | **Critical** | Cannot persist state | Yes |
| Missing frontend | **Critical** | Cannot display UI | Yes |
| Missing Docker/deploy | **Critical** | Cannot ship | Yes |
| Cholesky decomposition failure | **High** | Simulator crash | Yes |
| SSE streaming untested | **High** | Risk of data loss | No |
| No API integration tests | **High** | Risk of broken endpoints | Yes |
| No E2E tests | **High** | Risk of unusable product | No |
| No health check | **Medium** | Cannot monitor | No |
| Input validation missing | **Medium** | Security risk | No |
| Rate limiting missing | **Medium** | Cost/DDoS risk | No |

---

*Concerns audit: 2026-10-26*
