# Market Simulator

Approach and code structure for simulating realistic stock prices when no Massive API key is configured. Used as the default in FinAlly for development, demo, and free-tier deployments.

---

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** — the same stochastic process underlying Black-Scholes option pricing. Prices evolve with random noise, can never go negative, and produce the lognormal distribution observed in real equity markets.

Key properties:
- Updates every **500ms** — produces a live, streaming feel
- **Correlated moves** — tech stocks move together, financials cluster separately
- **Per-ticker volatility** — TSLA is wilder than V
- **Random shocks** — occasional 2–5% sudden moves for drama
- **Realistic seed prices** — starts at prices representative of the default watchlist

---

## GBM Mathematics

At each time step, a price evolves as:

```
S(t + dt) = S(t) × exp((μ - σ²/2) × dt + σ × √dt × Z)
```

Where:
- `S(t)` — current price
- `μ` — annualized drift (expected return), e.g. `0.05` = 5%/year
- `σ` — annualized volatility, e.g. `0.25` = 25%/year
- `dt` — time step as fraction of a trading year
- `Z` — standard normal random variable drawn from N(0, 1)

### Time Step Calculation

For 500ms updates, with 252 trading days/year and 6.5 trading hours/day:

```
dt = 0.5 seconds / (252 × 6.5 × 3600 seconds/year)
   = 0.5 / 5,896,800
   ≈ 8.48e-8
```

This tiny `dt` produces sub-cent per-tick moves that accumulate realistically over a trading day.

### Why GBM

- Multiplicative (`exp(...)`) — prices never go negative
- Independent increments — each step is memoryless (like real markets)
- Produces the characteristic "jagged random walk" of equity prices
- Parameters (`μ`, `σ`) map directly to intuitive financial concepts

---

## Correlated Moves

Real stocks don't move independently. We use **Cholesky decomposition** of a correlation matrix to generate correlated random draws.

Given a correlation matrix `C`, compute `L = cholesky(C)`. For independent standard normals `Z_ind`:
```
Z_corr = L @ Z_ind
```

Each element of `Z_corr` is correlated according to `C` but still standard normal.

### Default Correlation Groups

| Group | Tickers | Within-group ρ | Notes |
|-------|---------|----------------|-------|
| Tech | AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX | 0.6 | Strong co-movement |
| Finance | JPM, V | 0.5 | Rate-sensitive |
| TSLA | TSLA | 0.3 with everything | Does its own thing |
| Cross-group | Any tech ↔ finance | 0.3 | Market-wide baseline |
| Unknown tickers | Any not in above | 0.3 with all | Safe default |

---

## Random Shock Events

Every simulator step, each ticker has a small probability of a sudden shock — a 2–5% move in either direction. This adds the visual drama of breaking news without modeling actual news.

```python
EVENT_PROBABILITY = 0.001   # ~0.1% per tick per ticker

if random.random() < EVENT_PROBABILITY:
    shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
    price *= (1 + shock)
```

With 10 tickers updating at 500ms, expect a shock somewhere in the watchlist roughly every **50 seconds** — frequent enough to be interesting, rare enough not to be distracting.

---

## Seed Prices and Per-Ticker Parameters

```python
# backend/app/market/seed_data.py

SEED_PRICES: dict[str, float] = {
    "AAPL":  190.0,
    "GOOGL": 175.0,
    "MSFT":  420.0,
    "AMZN":  185.0,
    "TSLA":  250.0,
    "NVDA":  800.0,
    "META":  500.0,
    "JPM":   195.0,
    "V":     280.0,
    "NFLX":  600.0,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # High vol, lower drift
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # High vol, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # Low vol (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # Low vol (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}
```

Tickers added dynamically (not in `SEED_PRICES`) start at a random price between $50–$300.

---

## Implementation

```python
# backend/app/market/gbm.py
import math
import random
import numpy as np

from .seed_data import SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS

TECH    = {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"}
FINANCE = {"JPM", "V"}

# Seconds per 500ms tick as fraction of a trading year
DT = 0.5 / (252 * 6.5 * 3600)


class GBMSimulator:
    """
    Generates correlated GBM price paths for multiple tickers.
    Call step() at regular intervals to advance all prices.
    """

    def __init__(
        self,
        tickers: list[str],
        dt: float = DT,
        event_probability: float = 0.001,
    ):
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self.add_ticker(ticker)

    def add_ticker(self, ticker: str) -> None:
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50, 300))
        self._params[ticker] = TICKER_PARAMS.get(ticker, DEFAULT_PARAMS)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker: str) -> None:
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    def get_price(self, ticker: str) -> float | None:
        return self._prices.get(ticker)

    def step(self) -> dict[str, float]:
        """Advance one time step. Returns {ticker: new_price} for all active tickers."""
        n = len(self._tickers)
        if n == 0:
            return {}

        # Generate correlated standard normals
        z_independent = np.random.standard_normal(n)
        z = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result: dict[str, float] = {}
        for i, ticker in enumerate(self._tickers):
            mu    = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift     = (mu - 0.5 * sigma ** 2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * float(z[i])
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random shock event
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)

            result[ticker] = round(self._prices[ticker], 2)

        return result

    # ---- Internal ----

    def _rebuild_cholesky(self) -> None:
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return

        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = rho
                corr[j, i] = rho

        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _correlation(t1: str, t2: str) -> float:
        if t1 in TECH and t2 in TECH:
            return 0.6
        if t1 in FINANCE and t2 in FINANCE:
            return 0.5
        return 0.3
```

---

## SimulatorDataSource (Async Wrapper)

`GBMSimulator` is pure synchronous math. `SimulatorDataSource` wraps it in an async loop that feeds the shared `PriceCache`.

```python
# backend/app/market/simulator.py
import asyncio
from .interface import MarketDataSource, PriceCache
from .gbm import GBMSimulator


class SimulatorDataSource(MarketDataSource):

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
                prices = self._sim.step()
                for ticker, price in prices.items():
                    self._cache.update(ticker=ticker, price=price)
            await asyncio.sleep(self._interval)
```

---

## File Structure

```
backend/
  app/
    market/
      __init__.py
      models.py         # PriceUpdate dataclass
      interface.py      # MarketDataSource ABC + PriceCache
      factory.py        # create_market_data_source()
      massive_client.py # MassiveDataSource (real API)
      simulator.py      # SimulatorDataSource (async wrapper)
      gbm.py            # GBMSimulator (pure math, no async)
      seed_data.py      # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS
```

`gbm.py` has no async dependencies and can be unit-tested without FastAPI or `asyncio`.

---

## Behavior Notes

- **Prices never go negative** — GBM is multiplicative (`exp(...)` is always positive)
- **Sub-cent moves per tick** — the tiny `dt` means individual steps are imperceptible; the cumulative drift and volatility emerge over minutes of simulated trading
- **TSLA's σ=0.50** produces roughly the right intraday range (±2–5%) over a simulated trading day
- **Cholesky rebuild is O(n²)** but `n < 50` tickers in practice — safe to rebuild on every add/remove
- **Correlation matrix is guaranteed positive semi-definite** when all off-diagonal values are ≤ 1 and the matrix is symmetric — Cholesky decomposition only succeeds on valid matrices, so invalid configs will raise a `LinAlgError` at startup
- **Unknown tickers** use `DEFAULT_PARAMS = {sigma: 0.25, mu: 0.05}` and correlation 0.3 with all existing tickers
- **Shocks accumulate with the GBM step** — the shock multiplier is applied after the GBM update, keeping the price path continuous
