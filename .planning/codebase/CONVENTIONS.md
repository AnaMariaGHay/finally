---
last_mapped_commit: b97d1fef40a1301a65818a1554fd2d67cc274d27
last_mapped_at: 2026-10-26
---
# Coding Conventions

**Analysis Date:** 2026-10-26

## Naming Patterns

**Files:**
- Module files use `snake_case`: `cache.py`, `interface.py`, `massive_client.py`
- Test files use `test_*.py` pattern: `test_cache.py`, `test_simulator.py`
- Package directories use `snake_case`: `app/market/`

**Classes:**
- Use `PascalCase` (CamelCase): `PriceUpdate`, `PriceCache`, `MarketDataSource`, `GBMSimulator`
- Abstract base classes use suffix or prefix pattern: `MarketDataSource` with `@abstractmethod`

**Functions and Methods:**
- Use `snake_case`: `update()`, `get_price()`, `add_ticker()`, `_poll_once()`, `get_all()`
- Private/internal methods prefixed with single underscore: `_add_ticker_internal()`, `_rebuild_cholesky()`, `_poll_loop()`

**Variables:**
- Use `snake_case`: `price`, `ticker`, `previous_price`, `timestamp`, `dt`, `event_prob`
- Constants in `UPPER_SNAKE_CASE`: `DEFAULT_DT`, `TRADING_SECONDS_PER_YEAR`, `INTRA_TECH_CORR`
- Private instance variables prefixed with underscore: `self._prices`, `self._tickers`, `self._lock`

**Types:**
- Use modern Python 3.10+ union syntax `X | Y` instead of `Union[X, Y]`
- Use `dict[str, float]` instead of `Dict[str, float]`
- Use `list[str]` instead of `List[str]`
- Optional values: `float | None` instead of `Optional[float]`

## Code Style

**Formatting:**
- Line length: 100 characters (configured in `pyproject.toml` under `tool.ruff`)
- Use 4-space indentation
- Automated formatting handled by Ruff

**Linting:**
- Tool: `ruff` (configured in `pyproject.toml`)
- Rules enabled: E (pycodestyle errors), F (Pyflakes), I (isort imports), N (pep8-naming), W (pycodestyle warnings)
- Line-too-long (E501) ignored in lint rules but checked by formatter
- Target version: Python 3.12

**Run linting:**

```bash
uv run --extra dev ruff check app/ tests/
```

## Import Organization

**Order:**
1. `from __future__ import annotations` (at top of file)
2. Standard library imports: `time`, `asyncio`, `logging`, `math`, `random`, `dataclasses`, `threading`, `os`
3. Third-party imports: `numpy`, `massive`, `fastapi`, `uvicorn`
4. Local imports: `from .cache import PriceCache`, `from .interface import MarketDataSource`

**Path Aliases:**
- Not detected in current codebase (imports use relative `.` style within packages)

**Example Import Block:**

```python
from __future__ import annotations

import asyncio
import logging
import math

import numpy as np

from .cache import PriceCache
from .interface import MarketDataSource
```

## Error Handling

**Patterns:**

**Specific Exception Catching** — Catch specific exception types before generic:

```python

# In massive_client.py:_poll_once()

try:
    price = snap.last_trade.price
    timestamp = snap.last_trade.timestamp / 1000.0
except (AttributeError, TypeError) as e:
    logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "???"), e)
except Exception as e:
    logger.error("Massive poll failed: %s", e)
```

**Non-Fatal Errors in Background Tasks** — Log and continue, don't crash:

```python

# From massive_client.py — poll loop catches all exceptions and logs, then retries

try:
    snapshots = await asyncio.to_thread(self._fetch_snapshots)
except Exception as e:
    logger.error("Massive poll failed: %s", e)
    # Don't re-raise — the loop will retry on the next interval
```

**Validation via Testing** — Code validates assumptions with test coverage rather than runtime assertions. Prefer failing tests over try/except for programming errors.

## Logging

**Framework:** `logging` module (Python standard library)

**Patterns:**

**Module-Level Logger:**

```python
logger = logging.getLogger(__name__)
```

**Log Levels:**
- `logger.debug()` — Development info, algorithm details: "Random event on AAPL: 2.1% up"
- `logger.info()` — Lifecycle events: "Massive poller started: 10 tickers, 15.0s interval"
- `logger.warning()` — Recoverable errors: "Skipping snapshot for AAPL: AttributeError"
- `logger.error()` — Significant failures: "Massive poll failed: network error"

**Usage Examples:**

```python
logger.info("Massive poller started: %d tickers, %.1fs interval", len(tickers), self._interval)
logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))
logger.warning("Skipping snapshot for %s: %s", getattr(snap, "ticker", "???"), e)
logger.error("Massive poll failed: %s", e)
```

## Comments

**When to Comment:**

**DO comment:**
- Complex mathematical logic: GBM formula, Cholesky decomposition reasoning
- Performance considerations: "This is the hot path — called every 500ms. Keep it fast."
- Time unit conversions: "Massive timestamps are Unix milliseconds → convert to seconds"
- Design decisions and constraints: "Lifecycle: source = create... await source.start..."
- Non-obvious behavior: "Exp() is always positive, so GBM prices never go negative"

**DON'T comment:**
- Self-explanatory code (e.g., `cache.update("AAPL", 190.50)`)
- What the code does (it's obvious from reading it)
- Temporary TODOs without context (use FIXME/TODO with explanation)

**Comment Style:**
- Use `#` for single-line comments
- Capitalize first word, punctuate for readability
- Inline comments aligned with code

## Documentation

**Module Docstrings:**

```python
"""GBM-based market simulator."""

"""Thread-safe in-memory price cache."""
```

**Class Docstrings:**

```python
class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # ... app runs ...
        await source.stop()
    """
```

**Method Docstrings:**

```python
def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
    """Record a new price for a ticker. Returns the created PriceUpdate.

    Automatically computes direction and change from the previous price.
    If this is the first update for the ticker, previous_price == price (direction='flat').
    """
```

**Property Docstrings:**

```python
@property
def change(self) -> float:
    """Absolute price change from previous update."""
    return round(self.price - self.previous_price, 4)
```

## Function Design

**Size:** Functions range from 5–30 lines. Longer functions (>50 lines) should be split:
- `GBMSimulator.step()` — ~40 lines, justified by tight loop performance
- `_poll_once()` — ~30 lines, single responsibility (poll and update cache)

**Parameters:**
- Use type hints for all parameters
- Avoid positional-only arguments in public APIs
- Max 5–6 parameters for readability; use dataclasses for complex objects
- Default parameters for optional settings: `update_interval=0.5`, `event_probability=0.001`

**Return Values:**
- Always specify return type hint
- Return value types: primitive (`float`, `str`, `bool`), dataclass (`PriceUpdate`), collection (`dict[str, float]`), or `None`
- Avoid returning tuples; use dataclasses instead

**Example:**

```python
def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
    """..."""
    with self._lock:
        ts = timestamp or time.time()
        prev = self._prices.get(ticker)
        previous_price = prev.price if prev else price
        update = PriceUpdate(ticker=ticker, price=round(price, 2), ...)
        self._prices[ticker] = update
        self._version += 1
        return update
```

## Module Design

**Exports:**
- Public classes and functions at module level (imported via `from module import Class`)
- Private helpers prefixed with `_` (not part of public API)
- Each module has one primary public class or function

**Barrel Files:**
- `__init__.py` files use explicit `__all__` to document public API:

```python

# app/market/__init__.py

from .cache import PriceCache, PriceUpdate
from .factory import create_market_data_source
from .interface import MarketDataSource

__all__ = ["PriceCache", "PriceUpdate", "MarketDataSource", "create_market_data_source"]
```

**Example Module Structure:**

```
app/market/
├── __init__.py          # Exports public API
├── interface.py         # Abstract base classes
├── models.py            # Data classes
├── cache.py             # PriceCache implementation
├── simulator.py         # GBMSimulator implementation
├── massive_client.py    # MassiveDataSource implementation
├── factory.py           # create_market_data_source factory
├── seed_prices.py       # Configuration and seed data
├── stream.py            # SSE streaming router
└── tests/
    ├── conftest.py
    ├── test_*.py        # One test file per module
```

## Type Hints

**Always use type hints:**

```python
def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
    ...

def get_all(self) -> dict[str, PriceUpdate]:
    ...

def get_price(self, ticker: str) -> float | None:
    ...
```

**Use dataclasses for immutable data:**

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)
```

## Thread Safety

**Patterns:**
- Use `threading.Lock()` for shared mutable state
- Protect all access to shared state with `with self._lock:`
- No direct access to `self._prices` outside lock context

**Example:**

```python
class PriceCache:
    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()

    def update(self, ticker: str, price: float) -> PriceUpdate:
        with self._lock:
            prev = self._prices.get(ticker)
            # ... compute update ...
            self._prices[ticker] = update
            self._version += 1
            return update
```

---

*Convention analysis: 2026-10-26*
