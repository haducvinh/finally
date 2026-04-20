# Market Data Backend — Detailed Design

Complete design reference for the FinAlly market data subsystem. Covers the
unified interface, both concrete implementations (GBM simulator and Massive
API), the price cache, SSE streaming, and app lifecycle integration.

Status: **implemented** — all code in `backend/app/market/`. This document
serves as the authoritative design reference for that module.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Structure](#2-file-structure)
3. [Data Model — PriceUpdate](#3-data-model--priceupdate)
4. [Unified Interface — MarketDataSource](#4-unified-interface--marketdatasource)
5. [Price Cache](#5-price-cache)
6. [GBM Simulator](#6-gbm-simulator)
7. [SimulatorDataSource](#7-simulatordatasource)
8. [Massive API Client](#8-massive-api-client)
9. [Factory Function](#9-factory-function)
10. [SSE Streaming Endpoint](#10-sse-streaming-endpoint)
11. [App Lifecycle Integration](#11-app-lifecycle-integration)
12. [Downstream Usage Patterns](#12-downstream-usage-patterns)
13. [Testing Strategy](#13-testing-strategy)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Environment variable: MASSIVE_API_KEY                      │
│  ┌─────────────────────┐    ┌───────────────────────────┐   │
│  │  SimulatorDataSource│    │     MassiveDataSource     │   │
│  │  (default, no key)  │    │  (real data, key present) │   │
│  └──────────┬──────────┘    └─────────────┬─────────────┘   │
│             │                             │                  │
│             └─────────────┬───────────────┘                  │
│                           ▼                                  │
│                     PriceCache                               │
│             (thread-safe, in-memory)                         │
│                           │                                  │
│          ┌────────────────┼───────────────────┐             │
│          ▼                ▼                   ▼             │
│   SSE /api/stream   Portfolio          Trade Execution       │
│      /prices        Valuation          (price lookup)        │
└─────────────────────────────────────────────────────────────┘
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| Strategy pattern (ABC) | Downstream code never knows which source is active |
| PriceCache as single source of truth | One writer, many readers; no coupling between source and consumers |
| Push architecture | Source writes on its own schedule; consumers read when needed |
| Async-native | FastAPI event loop compatibility; blocking Massive SDK calls run in thread pool |
| SSE over WebSockets | One-way push is sufficient; simpler protocol, universal browser support |

---

## 2. File Structure

```
backend/
  app/
    market/
      __init__.py         # Public API surface — re-exports key types
      models.py           # PriceUpdate dataclass
      interface.py        # MarketDataSource abstract base class
      cache.py            # PriceCache — thread-safe price store
      seed_prices.py      # SEED_PRICES, TICKER_PARAMS, correlation constants
      simulator.py        # GBMSimulator (pure math) + SimulatorDataSource (async)
      massive_client.py   # MassiveDataSource — Polygon.io REST poller
      factory.py          # create_market_data_source() — selects source by env var
      stream.py           # create_stream_router() — FastAPI SSE endpoint
```

**Public API** (importable from `app.market`):

```python
from app.market import (
    PriceUpdate,              # models.py
    PriceCache,               # cache.py
    MarketDataSource,         # interface.py
    create_market_data_source, # factory.py
    create_stream_router,     # stream.py
)
```

---

## 3. Data Model — PriceUpdate

`PriceUpdate` is the **only** data structure that leaves the market layer. All
downstream code — SSE events, portfolio valuation, trade execution — works
exclusively with this type.

```python
# backend/app/market/models.py

from __future__ import annotations
import time
from dataclasses import dataclass, field


@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

**Design notes:**

- `frozen=True` — immutable; safe to pass across threads without copying
- `slots=True` — memory-efficient; lower overhead than `__dict__`-based objects
- `change` and `change_percent` are computed properties (not stored fields) — always consistent
- `to_dict()` is the canonical serialization path; used by the SSE endpoint

**Example serialized output** (for one SSE event field):

```json
{
  "ticker": "AAPL",
  "price": 191.24,
  "previous_price": 191.18,
  "timestamp": 1713600000.423,
  "change": 0.06,
  "change_percent": 0.0314,
  "direction": "up"
}
```

---

## 4. Unified Interface — MarketDataSource

```python
# backend/app/market/interface.py

from __future__ import annotations
from abc import ABC, abstractmethod


class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        # app runs ...
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        # app shutting down ...
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers.

        Starts a background task. Must be called exactly once.
        """

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources.

        Safe to call multiple times.
        """

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also removes it from the PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

**Contract invariants:**

- `start()` must be called before `add_ticker()` / `remove_ticker()`
- `stop()` is idempotent — safe to call even if never started
- `get_tickers()` is synchronous — no `await` needed (safe to call anytime)
- After `remove_ticker()`, the price is removed from `PriceCache` immediately

---

## 5. Price Cache

The cache is the single shared state between producers (data sources) and
consumers (SSE, portfolio, trades). No consumer ever calls the data source
directly.

```python
# backend/app/market/cache.py

from __future__ import annotations
import time
from threading import Lock
from .models import PriceUpdate


class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Monotonically increasing; bumped on every update

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Record a new price for a ticker. Returns the created PriceUpdate.

        Automatically computes direction and change from the previous price.
        If this is the first update for the ticker, previous_price == price (direction='flat').
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        """Get the latest price for a single ticker, or None if unknown."""
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all current prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def get_price(self, ticker: str) -> float | None:
        """Convenience: get just the price float, or None."""
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        """Remove a ticker from the cache."""
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Current version counter. Useful for SSE change detection."""
        return self._version

    def __len__(self) -> int:
        with self._lock:
            return len(self._prices)

    def __contains__(self, ticker: str) -> bool:
        with self._lock:
            return ticker in self._prices
```

**Version counter — why it matters for SSE:**

The `version` property increments on every `update()` call. The SSE generator
compares the current version to the last-sent version before deciding whether
to send a new event. This avoids emitting duplicate events when no price has
changed (important when using the Massive free tier, which polls every 15
seconds — the cache will be identical between polls).

```python
# SSE loop sketch
last_version = -1
while True:
    current_version = price_cache.version
    if current_version != last_version:
        last_version = current_version
        # send event
    await asyncio.sleep(0.5)
```

**Usage examples:**

```python
cache = PriceCache()

# Writer (called by data source background task)
update = cache.update("AAPL", 191.24)          # returns PriceUpdate
update = cache.update("AAPL", 191.30, ts=1713600000.5)  # with explicit timestamp

# Readers
price = cache.get_price("AAPL")          # float | None — quick price lookup
update = cache.get("AAPL")              # PriceUpdate | None — full snapshot
all_prices = cache.get_all()            # dict[str, PriceUpdate] — safe copy
"AAPL" in cache                         # bool — O(1) membership check
len(cache)                              # int — number of tracked tickers

# Cleanup
cache.remove("GOOGL")                   # evict from cache
```

---

## 6. GBM Simulator

`GBMSimulator` is pure synchronous math with no async dependencies. It can be
unit-tested independently of FastAPI or asyncio. `SimulatorDataSource` wraps
it in an async loop (see §7).

### 6.1 Mathematics

At each time step, every price evolves as:

```
S(t + dt) = S(t) × exp((μ - σ²/2) × dt + σ × √dt × Z)
```

| Symbol | Meaning | Typical value |
|--------|---------|---------------|
| `S(t)` | Current price | e.g. 191.24 |
| `μ` | Annualized drift (expected return) | 0.05 (5%/year) |
| `σ` | Annualized volatility | 0.22–0.50 depending on ticker |
| `dt` | Time step as fraction of a trading year | ~8.48e-8 for 500ms ticks |
| `Z` | Standard normal random variable | N(0,1) — correlated across tickers |

**Why GBM?**
- Multiplicative (`exp(...)` is always positive) — prices never go negative
- Independent increments — each step is memoryless, like real equity prices
- Parameters (`μ`, `σ`) map to standard financial concepts

**Time step calculation:**

```python
# 252 trading days/year × 6.5 hours/day × 3600 seconds/hour
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # = 5,896,800

# 500ms expressed as fraction of a trading year
DT = 0.5 / TRADING_SECONDS_PER_YEAR           # ≈ 8.48e-8
```

This tiny `dt` produces sub-cent moves per tick. Volatility and drift
accumulate naturally over simulated trading time.

### 6.2 Correlated Moves

Real stocks don't move independently. The simulator uses **Cholesky
decomposition** of a correlation matrix to generate correlated normal draws.

Given correlation matrix `C`, compute `L = cholesky(C)`.
For independent draws `Z_ind ~ N(0,I)`:

```
Z_corr = L @ Z_ind
```

Each element of `Z_corr` is still standard normal but correlated per `C`.

**Default correlation structure:**

| Group | Tickers | Intra-group ρ | Notes |
|-------|---------|---------------|-------|
| Tech | AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX | 0.6 | Strong co-movement |
| Finance | JPM, V | 0.5 | Rate-sensitive cluster |
| TSLA | TSLA | 0.3 with everything | Does its own thing |
| Cross-sector | any tech ↔ finance | 0.3 | Market-wide baseline |
| Unknown | dynamically added tickers | 0.3 with all | Safe default |

```python
# backend/app/market/seed_prices.py

CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR    = 0.6
INTRA_FINANCE_CORR = 0.5
CROSS_GROUP_CORR   = 0.3
TSLA_CORR          = 0.3
```

### 6.3 Random Shock Events

Every tick, each ticker has a 0.1% chance of a sudden 2–5% move:

```python
EVENT_PROBABILITY = 0.001

if random.random() < EVENT_PROBABILITY:
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    price *= (1 + shock_magnitude * shock_sign)
```

With 10 tickers at 2 ticks/second, expect a shock roughly every **50 seconds**
— frequent enough to feel lively, rare enough not to be distracting.

### 6.4 Seed Prices and Per-Ticker Parameters

```python
# backend/app/market/seed_prices.py

SEED_PRICES: dict[str, float] = {
    "AAPL":  190.00,
    "GOOGL": 175.00,
    "MSFT":  420.00,
    "AMZN":  185.00,
    "TSLA":  250.00,
    "NVDA":  800.00,
    "META":  500.00,
    "JPM":   195.00,
    "V":     280.00,
    "NFLX":  600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},  # High volatility, lower drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},  # High volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},  # Low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},  # Low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}
```

Dynamically added tickers not in `SEED_PRICES` start at `random.uniform(50, 300)`.

### 6.5 Implementation

```python
# backend/app/market/simulator.py — GBMSimulator class

import math, random, logging
import numpy as np
from .seed_prices import (
    CORRELATION_GROUPS, CROSS_GROUP_CORR, DEFAULT_PARAMS,
    INTRA_FINANCE_CORR, INTRA_TECH_CORR, SEED_PRICES, TICKER_PARAMS, TSLA_CORR,
)

logger = logging.getLogger(__name__)


class GBMSimulator:
    """Generates correlated GBM price paths for multiple tickers."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR  # ~8.48e-8

    def __init__(
        self,
        tickers: list[str],
        dt: float = DEFAULT_DT,
        event_probability: float = 0.001,
    ) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers one time step. Returns {ticker: new_price}."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu    = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift     = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * float(z_correlated[i])
            self._prices[ticker] *= math.exp(drift + diffusion)

            if random.random() < self._event_prob:
                shock_magnitude = random.uniform(0.02, 0.05)
                shock_sign = random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock_magnitude * shock_sign

            result[ticker] = round(self._prices[ticker], 2)

        return result

    def add_ticker(self, ticker: str) -> None:
        """Add a ticker. Rebuilds the correlation matrix."""
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Rebuilds the correlation matrix."""
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internals ---

    def _add_ticker_internal(self, ticker: str) -> None:
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self) -> None:
        """O(n²) but n < 50 tickers in practice — safe on add/remove."""
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
        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1: str, t2: str) -> float:
        tech    = CORRELATION_GROUPS["tech"]
        finance = CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

**Usage example (standalone, no async needed):**

```python
from app.market.simulator import GBMSimulator

sim = GBMSimulator(["AAPL", "TSLA", "JPM"])

for _ in range(5):
    prices = sim.step()   # {"AAPL": 190.12, "TSLA": 249.87, "JPM": 195.03}
    print(prices)

sim.add_ticker("NVDA")    # dynamically add — Cholesky rebuilt
sim.remove_ticker("JPM")  # dynamically remove — Cholesky rebuilt
```

---

## 7. SimulatorDataSource

`SimulatorDataSource` wraps `GBMSimulator` in an asyncio task that writes
to `PriceCache` every 500ms. It implements `MarketDataSource`.

```python
# backend/app/market/simulator.py — SimulatorDataSource class

import asyncio, logging
from .cache import PriceCache
from .interface import MarketDataSource
from .simulator import GBMSimulator  # same file; shown separately for clarity

logger = logging.getLogger(__name__)


class SimulatorDataSource(MarketDataSource):
    """MarketDataSource backed by the GBM simulator."""

    def __init__(
        self,
        price_cache: PriceCache,
        update_interval: float = 0.5,
        event_probability: float = 0.001,
    ) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        # Seed the cache immediately so SSE has data before the first tick
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")
        logger.info("Simulator started with %d tickers", len(tickers))

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        logger.info("Simulator stopped")

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            # Seed the cache immediately so the ticker has a price at once
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
            logger.info("Simulator: added ticker %s", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)
        logger.info("Simulator: removed ticker %s", ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    prices = self._sim.step()
                    for ticker, price in prices.items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

**Immediate cache seeding** (in `start()` and `add_ticker()`) is important:
the cache is populated before the first 500ms tick fires, so SSE clients
and any API calls immediately after startup get valid prices.

---

## 8. Massive API Client

`MassiveDataSource` polls the Massive (Polygon.io) snapshot endpoint for all
watched tickers in a single API call. The Massive Python SDK is synchronous,
so each call runs via `asyncio.to_thread` to avoid blocking the event loop.

```python
# backend/app/market/massive_client.py

from __future__ import annotations
import asyncio, logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from .cache import PriceCache
from .interface import MarketDataSource

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """MarketDataSource backed by the Massive (Polygon.io) REST API.

    Polls GET /v2/snapshot/locale/us/markets/stocks/tickers for all watched
    tickers in a single API call per interval. This is the correct approach
    for staying within free-tier rate limits (5 calls/min).
    """

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,   # 15s = safe for free tier (5 req/min)
    ) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        # Immediate first poll so cache has data right away
        await self._poll_once()
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")
        logger.info(
            "Massive poller started: %d tickers, %.1fs interval",
            len(tickers), self._interval,
        )

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None
        self._client = None
        logger.info("Massive poller stopped")

    async def add_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        if ticker not in self._tickers:
            self._tickers.append(ticker)
            logger.info("Massive: added %s (appears on next poll)", ticker)

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper().strip()
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)
        logger.info("Massive: removed %s", ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # --- Internal ---

    async def _poll_loop(self) -> None:
        """Sleep first — start() already did the initial poll."""
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        """One poll cycle: fetch all snapshots, update cache."""
        if not self._tickers or not self._client:
            return
        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            processed = 0
            for snap in snapshots:
                try:
                    price     = snap.last_trade.price
                    timestamp = snap.last_trade.timestamp / 1000.0  # ms → seconds
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
                    processed += 1
                except (AttributeError, TypeError) as e:
                    logger.warning("Skipping snapshot for %s: %s",
                                   getattr(snap, "ticker", "???"), e)
            logger.debug("Massive poll: updated %d/%d tickers", processed, len(self._tickers))
        except Exception as e:
            logger.error("Massive poll failed: %s", e)
            # Don't re-raise — loop retries on next interval.
            # Common: 401 (bad key), 429 (rate limit), network errors.

    def _fetch_snapshots(self) -> list:
        """Synchronous SDK call. Runs in a thread via asyncio.to_thread."""
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

### Poll Interval Selection

| Tier | Price/mo | Calls/min | Recommended interval |
|------|----------|-----------|---------------------|
| Free | Free | 5 | **15s** (leaves headroom at 4 calls/min) |
| Starter | $29 | 5 | **15s** |
| Developer | $79 | 30–60 | **3s** |
| Professional | $199 | 120+ | **1–2s** |

### Massive Snapshot Response Fields Used

```python
snap.ticker                   # str: "AAPL"
snap.last_trade.price         # float: last traded price — used for display and trades
snap.last_trade.timestamp     # int: Unix milliseconds — divide by 1000 for seconds
snap.day.previous_close       # float: prior day's close — for day-change calculations
snap.day.change_percent       # float: daily % change
snap.day.open                 # float: day open (OHLC tooltip)
snap.day.high                 # float: day high
snap.day.low                  # float: day low
snap.day.volume               # int: day volume
```

### Error Handling

```python
from massive.exceptions import BadResponse

try:
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
except BadResponse as e:
    if e.status == 401:
        logger.error("Invalid MASSIVE_API_KEY — check environment variable")
    elif e.status == 429:
        logger.warning("Rate limited — increase poll_interval or upgrade tier")
    elif e.status == 403:
        logger.error("Endpoint not available on this tier")
    else:
        logger.error("Massive API error %d: %s", e.status, e)
```

The `MassiveDataSource` absorbs all exceptions internally and logs them. The
poll loop continues — transient errors (network glitches, 429s) resolve on the
next interval without crashing the server.

---

## 9. Factory Function

```python
# backend/app/market/factory.py

from __future__ import annotations
import logging, os
from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

logger = logging.getLogger(__name__)


def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Select and create the appropriate data source based on environment.

    - MASSIVE_API_KEY set and non-empty → MassiveDataSource (real market data)
    - Otherwise → SimulatorDataSource (GBM simulation, default)

    Returns an unstarted source. Caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        logger.info("Market data source: Massive API (real data)")
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        logger.info("Market data source: GBM Simulator")
        return SimulatorDataSource(price_cache=price_cache)
```

**Environment variable behavior:**

| `MASSIVE_API_KEY` value | Source selected |
|------------------------|-----------------|
| Not set | `SimulatorDataSource` |
| Set to empty string `""` | `SimulatorDataSource` |
| Set to any non-empty value | `MassiveDataSource` |

---

## 10. SSE Streaming Endpoint

The SSE endpoint is completely source-agnostic — it reads only from
`PriceCache`. It uses a **version counter** to avoid sending duplicate events
when prices haven't changed (e.g., between Massive API polls).

```python
# backend/app/market/stream.py

from __future__ import annotations
import asyncio, json, logging
from collections.abc import AsyncGenerator
from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse
from .cache import PriceCache

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/api/stream", tags=["streaming"])


def create_stream_router(price_cache: PriceCache) -> APIRouter:
    """Factory: create the SSE router bound to a specific PriceCache instance."""

    @router.get("/prices")
    async def stream_prices(request: Request) -> StreamingResponse:
        """SSE endpoint: streams all tracked ticker prices every ~500ms.

        Client connects with EventSource. Each message is:
            data: {"AAPL": {...PriceUpdate fields...}, "GOOGL": {...}, ...}

        The retry directive tells the browser to reconnect after 1 second
        if the connection drops.
        """
        return StreamingResponse(
            _generate_events(price_cache, request),
            media_type="text/event-stream",
            headers={
                "Cache-Control": "no-cache",
                "Connection": "keep-alive",
                "X-Accel-Buffering": "no",  # Disable nginx buffering if proxied
            },
        )

    return router


async def _generate_events(
    price_cache: PriceCache,
    request: Request,
    interval: float = 0.5,
) -> AsyncGenerator[str, None]:
    """Yields SSE-formatted price events until client disconnects."""
    yield "retry: 1000\n\n"  # Tell browser to reconnect after 1s if dropped

    last_version = -1
    client_ip = request.client.host if request.client else "unknown"
    logger.info("SSE client connected: %s", client_ip)

    try:
        while True:
            if await request.is_disconnected():
                logger.info("SSE client disconnected: %s", client_ip)
                break

            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()
                if prices:
                    data = {ticker: update.to_dict() for ticker, update in prices.items()}
                    yield f"data: {json.dumps(data)}\n\n"

            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        logger.info("SSE stream cancelled: %s", client_ip)
```

### SSE Event Format

Each event is a single `data:` line followed by two newlines:

```
retry: 1000

data: {"AAPL": {"ticker": "AAPL", "price": 191.24, "previous_price": 191.18, "timestamp": 1713600000.423, "change": 0.06, "change_percent": 0.0314, "direction": "up"}, "GOOGL": {...}, ...}

data: {"AAPL": {"ticker": "AAPL", "price": 191.31, ...}, ...}
```

### Frontend Connection (TypeScript)

```typescript
const eventSource = new EventSource('/api/stream/prices');

eventSource.onmessage = (event) => {
  const prices: Record<string, PriceUpdate> = JSON.parse(event.data);
  Object.values(prices).forEach(update => {
    updateTickerDisplay(update);
  });
};

eventSource.onerror = () => {
  // EventSource retries automatically after the retry: 1000 directive
  setConnectionStatus('reconnecting');
};
```

### Frontend Flash Animation

Flash only when the price **value** changes, not on every SSE message
(important when using the Massive free tier — the same price will be
re-emitted multiple times between 15-second polls):

```typescript
function updateTickerDisplay(update: PriceUpdate) {
  const el = document.getElementById(`price-${update.ticker}`);
  if (!el) return;

  const currentDisplayed = parseFloat(el.dataset.price ?? '0');
  if (update.price !== currentDisplayed) {
    el.dataset.price = String(update.price);
    el.textContent = update.price.toFixed(2);

    // Flash animation — add class, remove after transition
    el.classList.remove('flash-up', 'flash-down');
    void el.offsetWidth; // Force reflow to restart animation
    el.classList.add(update.direction === 'up' ? 'flash-up' : 'flash-down');
  }
}
```

---

## 11. App Lifecycle Integration

```python
# backend/app/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI
from .market import PriceCache, create_market_data_source, create_stream_router
from .db import get_watchlist_tickers  # reads default user's watchlist from SQLite

price_cache = PriceCache()
market_source = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    global market_source

    # Load initial tickers from the database watchlist
    initial_tickers = get_watchlist_tickers(user_id="default")

    # Start the market data source (simulator or Massive)
    market_source = create_market_data_source(price_cache)
    await market_source.start(initial_tickers)

    yield  # App runs here

    # Graceful shutdown
    await market_source.stop()


app = FastAPI(lifespan=lifespan)

# Register the SSE streaming router
app.include_router(create_stream_router(price_cache))
```

### Watchlist Route Integration

When a ticker is added or removed via the watchlist API, the market source
and cache must be updated **after** the database write succeeds:

```python
# backend/app/routes/watchlist.py (sketch)

from fastapi import APIRouter
from ..market import PriceCache
from ..market.interface import MarketDataSource

router = APIRouter(prefix="/api/watchlist")


@router.post("/")
async def add_ticker(
    body: AddTickerRequest,
    market_source: MarketDataSource = Depends(get_market_source),
    db: sqlite3.Connection = Depends(get_db),
) -> dict:
    ticker = body.ticker.upper().strip()

    # 1. Write to DB first
    db.execute("INSERT OR IGNORE INTO watchlist (user_id, ticker) VALUES (?, ?)",
               ("default", ticker))
    db.commit()

    # 2. Update market source and cache
    await market_source.add_ticker(ticker)

    return {"ticker": ticker, "added": True}


@router.delete("/{ticker}")
async def remove_ticker(
    ticker: str,
    market_source: MarketDataSource = Depends(get_market_source),
    db: sqlite3.Connection = Depends(get_db),
) -> dict:
    ticker = ticker.upper().strip()

    # 1. Remove from DB
    db.execute("DELETE FROM watchlist WHERE user_id = ? AND ticker = ?",
               ("default", ticker))
    db.commit()

    # 2. Remove from market source and cache (cache.remove is called internally)
    await market_source.remove_ticker(ticker)

    return {"ticker": ticker, "removed": True}
```

### Trade Execution Price Lookup

```python
# backend/app/routes/portfolio.py (price lookup sketch)

@router.post("/trade")
async def execute_trade(
    body: TradeRequest,
    price_cache: PriceCache = Depends(get_price_cache),
    db: sqlite3.Connection = Depends(get_db),
) -> dict:
    ticker = body.ticker.upper().strip()
    price = price_cache.get_price(ticker)

    if price is None:
        raise HTTPException(status_code=400,
                            detail=f"No price available for {ticker}. "
                                   "Add it to your watchlist first.")

    # ... validate funds/shares, execute trade, update positions ...
```

---

## 12. Downstream Usage Patterns

### Quick Reference

```python
from app.market import PriceCache, create_market_data_source

# --- Startup ---
cache = PriceCache()
source = create_market_data_source(cache)      # reads MASSIVE_API_KEY env var
await source.start(["AAPL", "GOOGL", "MSFT"]) # begins background task

# --- Reading prices ---
update = cache.get("AAPL")           # PriceUpdate | None
price  = cache.get_price("AAPL")     # float | None — quick lookup
all_prices = cache.get_all()         # dict[str, PriceUpdate] — safe snapshot

# --- Serialize for API response ---
if update:
    response_data = update.to_dict() # dict ready for json.dumps / FastAPI

# --- Dynamic watchlist ---
await source.add_ticker("TSLA")      # adds to source + seeds cache immediately
await source.remove_ticker("GOOGL")  # removes from source + evicts from cache

# --- Introspection ---
tickers = source.get_tickers()       # list[str] — currently tracked tickers
version = cache.version              # int — increments on every price update
"AAPL" in cache                      # bool — O(1) membership check

# --- Shutdown ---
await source.stop()
```

### Dependency Injection Pattern (FastAPI)

```python
# backend/app/dependencies.py

from functools import lru_cache
from .market import PriceCache, create_market_data_source, MarketDataSource

_price_cache: PriceCache | None = None
_market_source: MarketDataSource | None = None


def get_price_cache() -> PriceCache:
    assert _price_cache is not None, "App not started"
    return _price_cache


def get_market_source() -> MarketDataSource:
    assert _market_source is not None, "App not started"
    return _market_source
```

---

## 13. Testing Strategy

### Unit Tests (73 tests, all passing)

```
backend/tests/market/
  test_models.py          # 11 tests — PriceUpdate properties and to_dict()
  test_cache.py           # 13 tests — thread safety, version, get/update/remove
  test_simulator.py       # 17 tests — GBM math, add/remove ticker, Cholesky
  test_simulator_source.py # 10 tests — async lifecycle, start/stop/add/remove
  test_factory.py         #  7 tests — env var selection logic
  test_massive.py         # 13 tests — mock SDK, poll_once, error handling
```

### Example: Testing the Cache

```python
# tests/market/test_cache.py

import time
from app.market.cache import PriceCache


def test_first_update_is_flat():
    cache = PriceCache()
    update = cache.update("AAPL", 190.00)
    assert update.direction == "flat"        # no previous price
    assert update.previous_price == 190.00
    assert update.change == 0.0


def test_direction_detection():
    cache = PriceCache()
    cache.update("AAPL", 190.00)
    up   = cache.update("AAPL", 191.00)
    down = cache.update("AAPL", 189.00)
    assert up.direction == "up"
    assert down.direction == "down"


def test_version_increments():
    cache = PriceCache()
    v0 = cache.version
    cache.update("AAPL", 190.00)
    v1 = cache.version
    cache.update("AAPL", 191.00)
    v2 = cache.version
    assert v1 > v0
    assert v2 > v1


def test_remove_evicts_ticker():
    cache = PriceCache()
    cache.update("AAPL", 190.00)
    assert "AAPL" in cache
    cache.remove("AAPL")
    assert "AAPL" not in cache
    assert cache.get("AAPL") is None
```

### Example: Testing the Simulator

```python
# tests/market/test_simulator.py

import pytest
from app.market.simulator import GBMSimulator


def test_step_returns_all_tickers():
    sim = GBMSimulator(["AAPL", "GOOGL", "MSFT"])
    prices = sim.step()
    assert set(prices.keys()) == {"AAPL", "GOOGL", "MSFT"}


def test_prices_stay_positive():
    sim = GBMSimulator(["TSLA"])  # High volatility ticker
    for _ in range(1000):
        prices = sim.step()
        assert prices["TSLA"] > 0


def test_add_remove_ticker():
    sim = GBMSimulator(["AAPL"])
    sim.add_ticker("GOOGL")
    assert "GOOGL" in sim.get_tickers()
    sim.remove_ticker("AAPL")
    assert "AAPL" not in sim.get_tickers()
    prices = sim.step()
    assert "GOOGL" in prices
    assert "AAPL" not in prices
```

### Example: Testing MassiveDataSource (mocked)

```python
# tests/market/test_massive.py

import asyncio
from unittest.mock import MagicMock, patch, AsyncMock
import pytest
from app.market.cache import PriceCache
from app.market.massive_client import MassiveDataSource


def make_mock_snapshot(ticker: str, price: float) -> MagicMock:
    snap = MagicMock()
    snap.ticker = ticker
    snap.last_trade.price = price
    snap.last_trade.timestamp = 1713600000000  # Unix ms
    return snap


@pytest.mark.asyncio
async def test_poll_updates_cache():
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache, poll_interval=999)
    source._client = MagicMock()
    source._client.get_snapshot_all.return_value = [
        make_mock_snapshot("AAPL", 191.24),
        make_mock_snapshot("GOOGL", 174.56),
    ]

    source._tickers = ["AAPL", "GOOGL"]
    await source._poll_once()

    assert cache.get_price("AAPL") == 191.24
    assert cache.get_price("GOOGL") == 174.56


@pytest.mark.asyncio
async def test_malformed_snapshot_is_skipped():
    cache = PriceCache()
    source = MassiveDataSource(api_key="test-key", price_cache=cache)
    source._client = MagicMock()

    bad_snap = MagicMock()
    bad_snap.ticker = "AAPL"
    bad_snap.last_trade = None  # Will raise AttributeError on .price

    source._client.get_snapshot_all.return_value = [bad_snap]
    source._tickers = ["AAPL"]

    # Should not raise — bad snapshot is logged and skipped
    await source._poll_once()
    assert cache.get("AAPL") is None
```

### Running Tests

```bash
cd backend
uv run --extra dev pytest tests/market/ -v          # market tests only
uv run --extra dev pytest tests/market/ --cov=app   # with coverage
uv run --extra dev pytest -x                        # stop on first failure
```

---

## Appendix: Demo Script

A Rich terminal dashboard is available to verify the simulator visually:

```bash
cd backend
uv run market_data_demo.py
```

Displays a live-updating table with all 10 tickers, price sparklines, color-coded
direction arrows, and a log of notable price events. Runs for 60 seconds or until
`Ctrl+C`.

This is also useful for smoke-testing the `MassiveDataSource` — set
`MASSIVE_API_KEY` in `.env` and the demo will show real prices:

```bash
MASSIVE_API_KEY=your_key uv run market_data_demo.py
```
