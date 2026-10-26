---
last_mapped_commit: b97d1fef40a1301a65818a1554fd2d67cc274d27
last_mapped_at: 2026-10-26
---
# Testing Patterns

**Analysis Date:** 2026-10-26

## Test Framework

**Runner:**
- `pytest` (version >=8.3.0)
- Config: `pyproject.toml` under `[tool.pytest.ini_options]`

**Async Support:**
- `pytest-asyncio` (version >=0.24.0)
- Async mode: "auto" (enabled in `pyproject.ini_options`)
- Default event loop scope: "function" (one loop per test)

**Coverage:**
- `pytest-cov` (version >=5.0.0)
- Source: `app/` (configured in `[tool.coverage.run]`)
- Omit: `tests/*`

**Run Commands:**

```bash
uv run --extra dev pytest -v              # All tests, verbose
uv run --extra dev pytest --cov=app       # With coverage report
uv run --extra dev pytest -k test_update  # Run specific tests by name pattern
uv run --extra dev pytest tests/market/   # Run specific directory
```

## Test File Organization

**Location:**
- Tests co-located with source in `tests/` directory (parallel to `app/`)
- Structure mirrors source structure: `tests/market/test_*.py` mirrors `app/market/`

**Naming Convention:**
- Test files: `test_*.py`
- Test classes: `Test*` (e.g., `TestPriceCache`, `TestGBMSimulator`, `TestFactory`)
- Test functions: `test_*` (e.g., `test_update_and_get`, `test_prices_are_positive`)

**Directory Structure:**

```
backend/
├── app/
│   └── market/
│       ├── models.py
│       ├── cache.py
│       ├── simulator.py
│       └── ...
├── tests/
│   ├── conftest.py              # Shared fixtures
│   ├── market/
│   │   ├── __init__.py
│   │   ├── test_models.py       # Tests for models.py
│   │   ├── test_cache.py        # Tests for cache.py
│   │   ├── test_simulator.py    # Tests for simulator.py
│   │   ├── test_simulator_source.py
│   │   ├── test_factory.py
│   │   └── test_massive.py
```

## Test Structure

**Test Class Organization:**

```python
class TestPriceCache:
    """Unit tests for the PriceCache."""

    def test_update_and_get(self):
        """Test updating and getting a price."""
        # Setup
        cache = PriceCache()
        
        # Execute
        update = cache.update("AAPL", 190.50)
        result = cache.get("AAPL")
        
        # Assert
        assert update.ticker == "AAPL"
        assert result == update
```

**Test Suite Organization:**
- One `TestClassName` per public class being tested
- Group related test methods within the class
- Order tests logically (basic functionality → edge cases → error handling)

**Docstring Pattern:**

```python
def test_change_percent_up(self):
    """Test percentage change calculation (up)."""
    # One sentence describing what is being tested
```

## Async Testing

**Marking Async Tests:**

```python
@pytest.mark.asyncio
class TestSimulatorDataSource:
    """Integration tests for SimulatorDataSource."""

    async def test_start_populates_cache(self):
        """Test that start() immediately populates the cache."""
        cache = PriceCache()
        source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
        
        await source.start(["AAPL", "GOOGL"])
        assert cache.get("AAPL") is not None
        
        await source.stop()  # Always cleanup
```

**Cleanup Pattern:**

```python
async def test_add_ticker(self):
    """Test adding a ticker dynamically."""
    cache = PriceCache()
    source = SimulatorDataSource(price_cache=cache, update_interval=0.1)
    await source.start(["AAPL"])
    
    await source.add_ticker("TSLA")
    assert "TSLA" in source.get_tickers()
    
    await source.stop()  # Always cleanup at end
```

**Timing Tests:**

```python
async def test_prices_update_over_time(self):
    """Test that prices are updated periodically."""
    cache = PriceCache()
    source = SimulatorDataSource(price_cache=cache, update_interval=0.05)
    await source.start(["AAPL"])
    
    initial_version = cache.version
    await asyncio.sleep(0.3)  # Wait for several cycles
    
    assert cache.version > initial_version
    await source.stop()
```

## Mocking

**Framework:** `unittest.mock` (Python standard library)

**Common Mocking Patterns:**

**Mock Object Creation via Helper Function:**

```python
def _make_snapshot(ticker: str, price: float, timestamp_ms: int) -> MagicMock:
    """Create a mock Massive snapshot object."""
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade = MagicMock()
    snap.last_trade.price = price
    snap.last_trade.timestamp = timestamp_ms
    return snap
```

**Patch Environment Variables:**

```python
from unittest.mock import patch
import os

def test_creates_simulator_when_no_api_key(self):
    """Test that simulator is created when MASSIVE_API_KEY is not set."""
    cache = PriceCache()
    
    with patch.dict(os.environ, {}, clear=True):
        source = create_market_data_source(cache)
    
    assert isinstance(source, SimulatorDataSource)
```

**Patch Object Methods:**

```python
async def test_poll_updates_cache(self):
    """Test that polling updates the cache."""
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache, poll_interval=60.0)
    source._tickers = ["AAPL", "GOOGL"]
    source._client = MagicMock()
    
    mock_snapshots = [
        _make_snapshot("AAPL", 190.50, 1707580800000),
        _make_snapshot("GOOGL", 175.25, 1707580800000),
    ]
    
    with patch.object(source, "_fetch_snapshots", return_value=mock_snapshots):
        await source._poll_once()
    
    assert cache.get_price("AAPL") == 190.50
```

**Mock with Side Effects (Exceptions):**

```python
async def test_api_error_does_not_crash(self):
    """Test that API errors don't crash the poller."""
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache)
    source._tickers = ["AAPL"]
    source._client = MagicMock()
    
    with patch.object(source, "_fetch_snapshots", side_effect=Exception("network error")):
        await source._poll_once()  # Should not raise
    
    assert cache.get_price("AAPL") is None  # No update happened
```

**Verify Mock Was Called:**

```python
async def test_empty_tickers_skips_poll(self):
    """Test that polling is skipped when there are no tickers."""
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache)
    source._tickers = []
    
    with patch.object(source, "_fetch_snapshots") as mock_fetch:
        await source._poll_once()
        mock_fetch.assert_not_called()
```

**What to Mock:**
- External APIs and services (Massive, OpenRouter)
- Filesystem operations
- Environment variables
- Time-dependent behavior (using `asyncio.sleep` instead of actual waiting)
- Private internal state for isolation

**What NOT to Mock:**
- Code under test (test real logic)
- Standard library functions (logging, collections)
- Simple dataclasses (create real instances)
- The cache itself (test it directly)

## Fixtures and Test Data

**Shared Fixtures (conftest.py):**

```python

# backend/tests/conftest.py

import pytest
import asyncio

@pytest.fixture
def event_loop_policy():
    """Use the default event loop policy for all async tests."""
    return asyncio.DefaultEventLoopPolicy()
```

**Inline Fixtures:**
Most tests create their own simple fixtures:

```python
def test_update_and_get(self):
    """Test updating and getting a price."""
    cache = PriceCache()  # Inline fixture
    update = cache.update("AAPL", 190.50)
    assert update.ticker == "AAPL"
```

**Helper Functions for Test Data:**

```python
def _make_snapshot(ticker: str, price: float, timestamp_ms: int) -> MagicMock:
    """Create a mock Massive snapshot object."""
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade = MagicMock()
    snap.last_trade.price = price
    snap.last_trade.timestamp = timestamp_ms
    return snap

# Usage

mock_snapshots = [
    _make_snapshot("AAPL", 190.50, 1707580800000),
    _make_snapshot("GOOGL", 175.25, 1707580800000),
]
```

## Test Types and Scope

**Unit Tests:**
- Single class in isolation
- Mock external dependencies
- Fast execution (<1s typically)
- Location: Most tests in `tests/market/test_*.py`
- Example: `TestPriceUpdate`, `TestGBMSimulator`, `TestFactory`

**Integration Tests:**
- Multiple components together
- May use real (in-process) implementations
- Slower execution (0.5–5s)
- Location: `TestSimulatorDataSource` (tests simulator + cache together)
- Example: `test_prices_update_over_time()` waits for real async operations

**E2E Tests:**
- Not yet implemented in backend
- Planned for `test/` directory (separate from `backend/tests/`)
- Uses Playwright + Docker

## Common Test Patterns

**State Transition Testing:**

```python
def test_direction_up(self):
    """Test price update with upward direction."""
    cache = PriceCache()
    cache.update("AAPL", 190.00)  # Setup: initial price
    update = cache.update("AAPL", 191.00)  # Execute: new price
    assert update.direction == "up"  # Assert: changed state
```

**Edge Case Testing:**

```python
def test_change_percent_zero_previous(self):
    """Test percentage change with zero previous price."""
    update = PriceUpdate(ticker="AAPL", price=100.00, previous_price=0.00, timestamp=123.0)
    assert update.change_percent == 0.0  # Guard against division by zero
```

**Stress Testing:**

```python
def test_prices_are_positive(self):
    """GBM prices can never go negative."""
    sim = GBMSimulator(tickers=["AAPL"])
    for _ in range(10_000):  # Many iterations
        prices = sim.step()
        assert prices["AAPL"] > 0  # Check invariant
```

**Idempotency Testing:**

```python
async def test_stop_is_idempotent(self):
    """Test that stop() can be called multiple times."""
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache)
    
    await source.stop()
    await source.stop()  # Should not raise
```

**Guard Condition Testing:**

```python
def test_remove_nonexistent_is_noop(self):
    """Test that removing a non-existent ticker is a no-op."""
    cache = PriceCache()
    cache.remove("AAPL")  # Should not raise
```

**Normalization Testing:**

```python
async def test_add_ticker_uppercase_normalization(self):
    """Test that tickers are normalized to uppercase."""
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache)
    
    await source.add_ticker("aapl")
    assert "AAPL" in source.get_tickers()
```

**Exception Testing:**

```python
def test_immutability(self):
    """Test that PriceUpdate is immutable."""
    update = PriceUpdate(ticker="AAPL", price=190.50, previous_price=190.00, timestamp=1234567890.0)
    
    with pytest.raises(AttributeError):
        update.price = 200.00  # Should raise error
```

**Dunder Method Testing:**

```python
def test_len(self):
    """Test __len__ method."""
    cache = PriceCache()
    assert len(cache) == 0
    cache.update("AAPL", 190.00)
    assert len(cache) == 1

def test_contains(self):
    """Test __contains__ method."""
    cache = PriceCache()
    cache.update("AAPL", 190.00)
    assert "AAPL" in cache
    assert "GOOGL" not in cache
```

**Private State Verification:**

```python
def test_cholesky_rebuilds_on_add(self):
    """Test that Cholesky matrix is rebuilt when tickers are added."""
    sim = GBMSimulator(tickers=["AAPL"])
    assert sim._cholesky is None  # Only 1 ticker
    sim.add_ticker("GOOGL")
    assert sim._cholesky is not None  # Now 2 tickers, matrix exists
```

## Coverage

**Requirements:** No enforced minimum (configured in `tool.coverage.report`)

**View Coverage:**

```bash
uv run --extra dev pytest --cov=app --cov-report=html

# Opens coverage report in htmlcov/index.html

```

**Exclusion Rules (in pyproject.toml):**

```toml
[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
    "if TYPE_CHECKING:",
]
```

**Coverage of Key Components:**

| Component | Test File | Approach |
|-----------|-----------|----------|
| `PriceUpdate` | `test_models.py` | 100% unit tests |
| `PriceCache` | `test_cache.py` | 100% unit tests (thread safety, locking tested implicitly) |
| `GBMSimulator` | `test_simulator.py` | 100% unit tests (math properties, correlations, edge cases) |
| `SimulatorDataSource` | `test_simulator_source.py` | Integration tests with real async, timing checks |
| `MassiveDataSource` | `test_massive.py` | Mocked API, error handling, normalization |
| `Factory` | `test_factory.py` | Environment variable mocking, type checking |

## Running Tests

**All Tests:**

```bash
cd backend
uv run --extra dev pytest -v
```

**Specific Test Class:**

```bash
uv run --extra dev pytest -v tests/market/test_cache.py::TestPriceCache
```

**Specific Test Method:**

```bash
uv run --extra dev pytest -v tests/market/test_cache.py::TestPriceCache::test_update_and_get
```

**Watch Mode (requires pytest-watch):**

```bash
uv run --extra dev ptw tests/market/test_cache.py
```

**With Coverage:**

```bash
uv run --extra dev pytest --cov=app --cov-report=term-missing
```

---

*Testing analysis: 2026-10-26*
