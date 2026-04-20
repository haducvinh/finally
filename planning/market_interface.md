# Market Data Interface — Unified Python API

Design of the unified Python API for market data in FinAlly. Two concrete implementations (Massive API and simulator) sit behind one abstract interface. All downstream code — SSE streaming, trade execution, portfolio valuation — is source-agnostic.

---

## Design Goals

1. **Single abstraction** — the rest of the app never knows whether prices come from Massive or the simulator
2. **Push architecture** — the data source writes to a shared `PriceCache`; consumers read from it on their own schedule
3. **Async-native** — designed for FastAPI's async event loop; blocking Massive SDK calls are offloaded to a thread pool
4. **Live watchlist** — tickers can be added and removed at runtime without restarting the source

---

## Core Data Model

```python
# backend/app/market/models.py
from dataclasses import dataclass

@dataclass
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float        # Unix seconds
    change: float           # price - previous_price
    direction: str          # "up", "down", or "flat"
```

This is the **only** data structure that leaves the market layer. Everything downstream — SSE events, portfolio valuation, trade price lookup — works with `PriceUpdate`.

---

## Abstract Interface

```python
# backend/app/market/interface.py
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    """Abstract interface for market data providers."""

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates for the given tickers."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop producing price updates and clean up resources."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active watch set."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active watch set."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of active tickers."""
```

---

## Price Cache

Thread-safe in-memory store. The data source writes here; the SSE streamer and trade executor read from here.

```python
# backend/app/market/interface.py (continued)
import time
from threading import Lock

class PriceCache:
    """Thread-safe cache of latest prices per ticker."""

    def __init__(self):
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Update price for a ticker. Returns the resulting PriceUpdate."""
        with self._lock:
            ts = timestamp or time.time()
            previous = self._prices.get(ticker)
            previous_price = previous.price if previous else price

            direction = "up" if price > previous_price else "down" if price < previous_price else "flat"

            update = PriceUpdate(
                ticker=ticker,
                price=price,
                previous_price=previous_price,
                timestamp=ts,
                change=round(price - previous_price, 4),
                direction=direction,
            )
            self._prices[ticker] = update
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        with self._lock:
            return dict(self._prices)

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)
```

---

## Factory Function

Selects the data source at startup based on whether `MASSIVE_API_KEY` is set.

```python
# backend/app/market/factory.py
import os

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        from .massive_client import MassiveDataSource
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        from .simulator import SimulatorDataSource
        return SimulatorDataSource(price_cache=price_cache)
```

---

## MassiveDataSource Implementation

Polls the Massive snapshot endpoint on a configurable interval. The Massive Python SDK is synchronous, so calls run in a thread pool via `asyncio.to_thread`.

```python
# backend/app/market/massive_client.py
import asyncio
import logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

from .interface import MarketDataSource, PriceCache

logger = logging.getLogger(__name__)


class MassiveDataSource(MarketDataSource):
    """Polls the Massive API for live stock prices."""

    def __init__(
        self,
        api_key: str,
        price_cache: PriceCache,
        poll_interval: float = 15.0,     # 15s = safe for free tier (5 req/min)
    ):
        self._client = RESTClient(api_key=api_key)
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None

    # ---- MarketDataSource interface ----

    async def start(self, tickers: list[str]) -> None:
        self._tickers = list(tickers)
        self._task = asyncio.create_task(self._poll_loop())
        logger.info(f"MassiveDataSource started, polling {len(tickers)} tickers every {self._interval}s")

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

    async def add_ticker(self, ticker: str) -> None:
        if ticker not in self._tickers:
            self._tickers.append(ticker)

    async def remove_ticker(self, ticker: str) -> None:
        self._tickers = [t for t in self._tickers if t != ticker]
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    # ---- Internal ----

    async def _poll_loop(self) -> None:
        while True:
            try:
                await self._poll_once()
            except Exception as e:
                logger.warning(f"Massive poll error: {e}")
            await asyncio.sleep(self._interval)

    async def _poll_once(self) -> None:
        if not self._tickers:
            return

        # Snapshot is one API call for all tickers — critical for rate limit compliance
        snapshots = await asyncio.to_thread(
            self._client.get_snapshot_all,
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )

        for snap in snapshots:
            try:
                price = snap.last_trade.price
                ts = snap.last_trade.timestamp / 1000  # ms -> seconds
                self._cache.update(ticker=snap.ticker, price=price, timestamp=ts)
            except (AttributeError, TypeError) as e:
                logger.warning(f"Skipping malformed snapshot for {snap.ticker}: {e}")
```

### Poll Interval Selection

| Tier | Recommended interval | Rationale |
|------|----------------------|-----------|
| Free | 15s | 5 req/min = 1 call per 12s; 15s leaves headroom |
| Starter | 15s | Same limit as free |
| Developer | 3s | 30 req/min; one call per watchlist poll |
| Professional | 1–2s | Near real-time; stay under 120 req/min |

---

## SimulatorDataSource Implementation

Wraps `GBMSimulator` (see `market_simulator.md`) in an async loop, pushing price updates every 500ms.

```python
# backend/app/market/simulator.py (partial — see market_simulator.md for GBMSimulator)
import asyncio
from .interface import MarketDataSource, PriceCache
from .gbm import GBMSimulator


class SimulatorDataSource(MarketDataSource):
    """Generates synthetic GBM price paths — used when no Massive API key is set."""

    def __init__(self, price_cache: PriceCache, update_interval: float = 0.5):
        self._cache = price_cache
        self._interval = update_interval
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers)
        self._task = asyncio.create_task(self._run_loop())

    async def stop(self) -> None:
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self) -> list[str]:
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self) -> None:
        while True:
            if self._sim:
                prices = self._sim.step()          # dict[str, float]
                for ticker, price in prices.items():
                    self._cache.update(ticker=ticker, price=price)
            await asyncio.sleep(self._interval)
```

---

## SSE Integration

The SSE endpoint reads from `PriceCache.get_all()` every 500ms and yields JSON events. It is completely source-agnostic.

```python
# backend/app/routes/stream.py
import json
import asyncio
from fastapi import Request
from fastapi.responses import StreamingResponse
from ..market.interface import PriceCache


async def _price_generator(request: Request, price_cache: PriceCache):
    while True:
        if await request.is_disconnected():
            break
        prices = price_cache.get_all()
        payload = {
            ticker: {
                "ticker": p.ticker,
                "price": p.price,
                "previous_price": p.previous_price,
                "change": p.change,
                "direction": p.direction,
                "timestamp": p.timestamp,
            }
            for ticker, p in prices.items()
        }
        yield f"data: {json.dumps(payload)}\n\n"
        await asyncio.sleep(0.5)


def price_stream_response(request: Request, price_cache: PriceCache) -> StreamingResponse:
    return StreamingResponse(
        _price_generator(request, price_cache),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",
        },
    )
```

---

## App Lifecycle

```python
# backend/app/main.py (sketch)
from contextlib import asynccontextmanager
from fastapi import FastAPI
from .market.interface import PriceCache
from .market.factory import create_market_data_source
from .db import get_watchlist_tickers

price_cache = PriceCache()
market_source = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    global market_source
    initial_tickers = get_watchlist_tickers(user_id="default")
    market_source = create_market_data_source(price_cache)
    await market_source.start(initial_tickers)
    yield
    await market_source.stop()


app = FastAPI(lifespan=lifespan)
```

Watchlist changes call `market_source.add_ticker()` / `market_source.remove_ticker()` after updating the database.

---

## File Structure

```
backend/
  app/
    market/
      __init__.py
      models.py             # PriceUpdate dataclass
      interface.py          # MarketDataSource ABC + PriceCache
      factory.py            # create_market_data_source()
      massive_client.py     # MassiveDataSource
      simulator.py          # SimulatorDataSource
      gbm.py                # GBMSimulator (pure math — no async)
      seed_data.py          # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS
```

`gbm.py` contains only synchronous math with no async dependencies — it can be unit-tested independently of the async loop.

---

## Contract Summary

| Concern | Owner | Notes |
|---------|-------|-------|
| Price computation | `GBMSimulator` or Massive API | Never called directly by app code |
| Price storage | `PriceCache` | Single shared instance, injected via dependency |
| Source selection | `create_market_data_source()` | Reads `MASSIVE_API_KEY` env var |
| Ticker lifecycle | `MarketDataSource.add/remove_ticker()` | Called by watchlist routes |
| Price consumption | SSE route + trade executor | Read from `PriceCache` only |
