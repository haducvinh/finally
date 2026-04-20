# FinAlly — End-to-End Backend Documentation

## Overview

FinAlly (Finance Ally) is a full-stack AI-powered trading workstation. This document explains the backend implementation in detail — every file, its purpose, how it works, how components interact, and how the backend connects to the frontend.

The backend is built with **FastAPI** (Python 3.12+), managed by **uv**, and runs behind **uvicorn**. The market data subsystem is the first completed component; this document covers both what is implemented today and the planned full architecture.

---

## Directory Structure

```
backend/
├── app/
│   ├── __init__.py                  # Package marker
│   └── market/                      # Market data subsystem (COMPLETE)
│       ├── __init__.py              # Public API exports for the subsystem
│       ├── models.py                # PriceUpdate immutable dataclass
│       ├── cache.py                 # Thread-safe in-memory price store
│       ├── interface.py             # Abstract base class for data sources
│       ├── simulator.py             # GBM simulator + SimulatorDataSource
│       ├── massive_client.py        # Polygon.io REST client + MassiveDataSource
│       ├── factory.py               # Environment-driven source selection
│       ├── stream.py                # FastAPI SSE streaming endpoint
│       └── seed_prices.py           # Default prices, volatility, correlations
├── tests/
│   ├── conftest.py                  # Pytest fixtures
│   └── market/
│       ├── test_models.py           # 11 tests for PriceUpdate
│       ├── test_cache.py            # 13 tests for PriceCache
│       ├── test_simulator.py        # 17 tests for GBMSimulator
│       ├── test_simulator_source.py # 10 integration tests for SimulatorDataSource
│       ├── test_factory.py          # 7 tests for factory function
│       └── test_massive.py          # 13 tests for MassiveDataSource (mocked)
├── market_data_demo.py              # Rich terminal live dashboard (dev tool)
├── pyproject.toml                   # Project metadata and dependencies
├── uv.lock                          # Pinned dependency lockfile
├── README.md                        # Quick-start guide
└── CLAUDE.md                        # Developer API reference
```

---

## File-by-File Explanation

### `app/market/__init__.py` — Public API Surface

**What it does:** Defines the public API of the `app.market` package. Any code outside this directory imports from here rather than from individual modules.

**Exports:**
```python
from app.market import (
    PriceUpdate,              # Data model
    PriceCache,               # In-memory store
    MarketDataSource,         # Abstract interface
    create_market_data_source, # Factory function
    create_stream_router,     # SSE endpoint factory
)
```

**Why it matters:** This is the integration point. When the rest of the backend (portfolio routes, watchlist routes, future main app) needs market data, they import from this single namespace. It enforces clean boundaries — internal module structure can change without breaking callers.

---

### `app/market/models.py` — `PriceUpdate` Data Model

**What it does:** Defines the canonical data structure that flows through the entire market data pipeline.

**Key type:** `PriceUpdate` — a frozen (immutable) dataclass:

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds (time.time())

    # Computed properties (not stored, derived on demand)
    @property
    def change(self) -> float: ...          # price - previous_price
    @property
    def change_percent(self) -> float: ...  # % change
    @property
    def direction(self) -> str: ...         # "up", "down", or "flat"

    def to_dict(self) -> dict: ...          # For JSON / SSE serialization
```

**Design choices:**
- `frozen=True` — prevents accidental mutation. Once a price snapshot is created, it cannot be modified. To update a price, you create a new `PriceUpdate` object.
- `slots=True` — uses `__slots__` internally, making the dataclass faster and more memory-efficient (important when creating hundreds of these per second).
- `timestamp` defaults to `time.time()` — so callers don't have to pass it unless using a custom timestamp (e.g., when replaying API timestamps from Massive).

**Used by:** `PriceCache` (stores these), `stream.py` (serializes via `to_dict()` for SSE), and future portfolio/trade routes (read prices from cache).

---

### `app/market/cache.py` — `PriceCache` (Thread-Safe Price Store)

**What it does:** Maintains the latest price for every tracked ticker in memory. Acts as the single source of truth for current prices across the entire backend.

**Architecture role:** The cache is the shared data bus between the price producer (simulator or Massive API) and all price consumers (SSE streaming, portfolio valuation, trade execution).

```
Producers:                    Cache:                  Consumers:
SimulatorDataSource  ──────▶  PriceCache  ◀──────▶  SSE streaming
MassiveDataSource    ──────▶  (in-memory)            Portfolio routes
                                                      Trade execution
                                                      Watchlist pricing
```

**Key methods:**

| Method | Description |
|--------|-------------|
| `update(ticker, price, timestamp=None)` | Record a new price. Auto-computes `previous_price` from last known value. Returns the new `PriceUpdate`. |
| `get(ticker)` | Return the latest `PriceUpdate` for a ticker, or `None`. |
| `get_price(ticker)` | Shortcut: return just the `float` price, or `None`. |
| `get_all()` | Snapshot of all current prices as `{ticker: PriceUpdate}`. Returns a shallow copy. |
| `remove(ticker)` | Evict a ticker (called when removed from watchlist). |
| `version` (property) | Monotonic integer that increments on every `update()` call. |

**Thread safety:** All methods acquire `threading.Lock` before reading/writing `self._prices`. This is critical because the price producer (background asyncio task) and consumers (HTTP request handlers) may run concurrently.

**Version counter mechanism:** The `version` property is key to efficient SSE streaming. Instead of sending prices on a fixed timer regardless of whether anything changed, the SSE generator checks if `version` increased since its last send. This means:
- No duplicate data is sent to the frontend
- SSE events are driven by actual price changes, not wall-clock time

**First update behavior:** When a ticker's price is updated for the first time, `previous_price` is set equal to `price`. This means `direction` is `"flat"` and `change` is `0.0` — a safe initial state that won't trigger spurious red/green flash effects on the frontend.

---

### `app/market/interface.py` — `MarketDataSource` Abstract Base Class

**What it does:** Defines the contract that all market data providers must implement. This is the Strategy Pattern — the rest of the system doesn't care whether prices come from simulation or a real API; both implementations expose the same interface.

**Abstract methods:**

| Method | Description |
|--------|-------------|
| `start(tickers: list[str])` | Begin producing prices for the given tickers. Starts background polling/simulation. |
| `stop()` | Stop the background task and release resources. Safe to call multiple times. |
| `add_ticker(ticker: str)` | Add a ticker to the active set. Takes effect on next update cycle. |
| `remove_ticker(ticker: str)` | Remove a ticker and evict it from the cache. |
| `get_tickers()` | Return the currently tracked tickers. |

**Lifecycle contract:**
```
create_market_data_source(cache)  →  source
await source.start(["AAPL", "GOOGL", ...])
# App running:
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")
# App shutting down:
await source.stop()
```

**Why abstract?** Future phases may add additional data sources (e.g., a WebSocket-based provider, a replay system for backtesting). The interface ensures they're drop-in replacements.

---

### `app/market/seed_prices.py` — Default Ticker Configuration

**What it does:** Provides the starting configuration for the 10 default tickers: seed prices, GBM simulation parameters, and sector correlation structure.

**Contents:**

**`SEED_PRICES`** — Starting prices for the 10 default tickers:
```python
{"AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00,
 "TSLA": 250.00, "NVDA": 800.00, "META": 500.00, "JPM": 195.00,
 "V": 280.00, "NFLX": 600.00}
```

**`TICKER_PARAMS`** — Per-ticker GBM parameters (`sigma` = annualized volatility, `mu` = annualized drift):
```python
"TSLA": {"sigma": 0.50, "mu": 0.03}  # High volatility, lower drift
"NVDA": {"sigma": 0.40, "mu": 0.08}  # High volatility, strong upward drift
"JPM":  {"sigma": 0.18, "mu": 0.04}  # Stable bank stock
"V":    {"sigma": 0.17, "mu": 0.04}  # Stable payments stock
```

**`CORRELATION_GROUPS`** — Sector groupings that drive correlated price moves:
```python
"tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"}
"finance": {"JPM", "V"}
```

**Correlation coefficients:**
- Tech stocks with each other: `0.6` (move together strongly)
- Finance stocks with each other: `0.5`
- TSLA with anything: `0.3` (does its own thing despite being "tech")
- Cross-sector or unknown tickers: `0.3`

**`DEFAULT_PARAMS`** — Used when a ticker is dynamically added but not in `TICKER_PARAMS`:
```python
{"sigma": 0.25, "mu": 0.05}
```

**Used by:** `GBMSimulator` constructor reads all of these to initialize prices and build the correlation matrix.

---

### `app/market/simulator.py` — GBM Simulator + `SimulatorDataSource`

This file contains two classes with distinct responsibilities:

#### `GBMSimulator` — The Math Engine

**What it does:** Simulates realistic stock price movement using **Geometric Brownian Motion (GBM)** with sector-correlated random draws.

**The formula:**
```
S(t + dt) = S(t) × exp((μ - σ²/2) × dt + σ × √dt × Z)
```
Where:
- `S(t)` = current price
- `μ` (mu) = annualized drift (expected return)
- `σ` (sigma) = annualized volatility
- `dt` = time step as fraction of a trading year (~8.48×10⁻⁸ for 500ms steps)
- `Z` = correlated standard normal random variable

**Why GBM?** It's the standard model used in the Black-Scholes framework. It guarantees prices stay positive (exponential), naturally produces the random walk behavior seen in real markets, and the parameters (mu, sigma) map intuitively to real-world volatility and return.

**Time step calculation:**
```
dt = 0.5 seconds / (252 trading days × 6.5 hours/day × 3600 sec/hour)
   = 0.5 / 5,896,800
   ≈ 8.48 × 10⁻⁸
```
This tiny dt means each 500ms tick produces sub-cent price movements, which accumulate naturally over time into realistic-looking price action.

**Correlated moves via Cholesky decomposition:**
Rather than generating independent random draws per ticker, the simulator uses a correlation matrix and Cholesky decomposition to produce correlated draws. This means when AAPL moves up, MSFT and GOOGL tend to move up too (ρ = 0.6 intra-tech), while JPM only loosely follows (ρ = 0.3 cross-sector).

```python
z_independent = np.random.standard_normal(n)   # n independent draws
z_correlated  = cholesky_matrix @ z_independent # apply correlation structure
```

**Random market events:** With probability 0.1% per ticker per tick, a sudden shock of 2–5% (up or down) is applied. With 10 tickers at 2 ticks/second, an event occurs roughly every 50 seconds — adding realistic drama without overwhelming the simulation.

**Dynamic ticker management:** When a ticker is added or removed, `_rebuild_cholesky()` reconstructs the full correlation matrix. This is O(n²) but acceptable since n < 50 and it only happens on watchlist changes, not in the hot path.

**Key methods:**

| Method | Description |
|--------|-------------|
| `step()` | Hot path. Advances all tickers one time step. Called every 500ms. Returns `{ticker: new_price}`. |
| `add_ticker(ticker)` | Add a new ticker and rebuild correlation matrix. |
| `remove_ticker(ticker)` | Remove a ticker and rebuild correlation matrix. |
| `get_price(ticker)` | Current price for a specific ticker. |

---

#### `SimulatorDataSource` — Async Wrapper

**What it does:** Bridges `GBMSimulator` (synchronous math) into the async `MarketDataSource` lifecycle. Runs an asyncio background task that calls `simulator.step()` every 500ms and writes results to `PriceCache`.

**Startup sequence:**
1. Instantiate `GBMSimulator` with the initial ticker list
2. Seed the `PriceCache` immediately with initial prices (so SSE has data from the first request)
3. Launch `asyncio.create_task(_run_loop())` — the background update loop

**The update loop:**
```python
async def _run_loop(self):
    while True:
        prices = self._sim.step()          # Generate new prices
        for ticker, price in prices.items():
            self._cache.update(ticker, price)  # Write to shared cache
        await asyncio.sleep(0.5)           # Wait 500ms
```

**Error resilience:** Any exception in `step()` is caught and logged — the loop continues rather than crashing the entire server.

**Stop behavior:** `stop()` cancels the asyncio task and awaits its cleanup, ensuring the loop is fully stopped before the method returns. Idempotent — safe to call multiple times.

---

### `app/market/massive_client.py` — `MassiveDataSource` (Real Market Data)

**What it does:** Implements `MarketDataSource` using the Massive (Polygon.io) REST API for real market prices.

**Architecture:** The `massive` package's `RESTClient` is synchronous. To avoid blocking the asyncio event loop, API calls are dispatched to a thread pool using `asyncio.to_thread()`.

**Polling loop:**
```
startup:  await _poll_once()  → immediate first data load
loop:     await asyncio.sleep(interval) → await _poll_once() → repeat
```

**API call:**
```python
def _fetch_snapshots(self) -> list:
    return self._client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=self._tickers,
    )
```
This fetches the full snapshot for all watched tickers in a single API call, minimizing rate limit usage.

**Data extraction:**
```python
price     = snap.last_trade.price
timestamp = snap.last_trade.timestamp / 1000.0  # Convert ms → seconds
cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)
```

**Error handling:** Three levels of resilience:
1. Per-snapshot `AttributeError`/`TypeError` — logs a warning and skips the malformed entry
2. Poll-level exception — logs the error, does not re-raise, retries on next interval
3. Task-level `CancelledError` — cleanly exits on `stop()`

**Rate limits:**
- Free tier: 5 calls/min → default `poll_interval=15.0` seconds
- Paid tiers: lower intervals (2–5s) can be configured

---

### `app/market/factory.py` — `create_market_data_source()`

**What it does:** Reads the `MASSIVE_API_KEY` environment variable and returns the appropriate `MarketDataSource` implementation. This is the Factory Pattern.

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

**Why a factory?** The main application startup code doesn't need to know which implementation is used. It just calls this function, gets a `MarketDataSource`, and calls `await source.start(tickers)`. The implementation is an environment-level decision, not a code-level one.

**Caller (future `main.py` or lifespan handler):**
```python
cache  = PriceCache()
source = create_market_data_source(cache)
await source.start(DEFAULT_TICKERS)
# ... app runs ...
await source.stop()
```

---

### `app/market/stream.py` — SSE Streaming Endpoint

**What it does:** Exposes a `GET /api/stream/prices` endpoint that streams live price updates to the browser using **Server-Sent Events (SSE)**.

**Why SSE over WebSockets?** Price data flows one-way (server → client). SSE is simpler, uses plain HTTP, and the browser's `EventSource` API handles reconnection automatically. No need for bidirectional WebSocket complexity.

**Router factory:**
```python
def create_stream_router(price_cache: PriceCache) -> APIRouter:
    # Returns a FastAPI APIRouter with /prices bound to the cache
```

The factory pattern avoids global state — the `PriceCache` is injected, making the endpoint testable and reusable.

**The SSE response:**
```python
return StreamingResponse(
    _generate_events(price_cache, request),
    media_type="text/event-stream",
    headers={
        "Cache-Control": "no-cache",
        "Connection": "keep-alive",
        "X-Accel-Buffering": "no",   # Disable nginx buffering if proxied
    },
)
```

**The event generator:**
```python
async def _generate_events(price_cache, request, interval=0.5):
    yield "retry: 1000\n\n"    # Tell browser: retry after 1 second if disconnected
    last_version = -1

    while True:
        if await request.is_disconnected():
            break                         # Client disconnected, stop

        current_version = price_cache.version
        if current_version != last_version:   # Only send when prices changed
            last_version = current_version
            prices = price_cache.get_all()
            if prices:
                data = {t: u.to_dict() for t, u in prices.items()}
                yield f"data: {json.dumps(data)}\n\n"

        await asyncio.sleep(interval)
```

**Key design decisions:**
- **Version-based change detection**: `last_version` tracks the last cache version sent. If nothing changed since the last send, no event is emitted — conserving bandwidth and avoiding spurious frontend re-renders.
- **500ms polling interval**: Balances real-time feel with CPU efficiency. Combined with the simulator's 500ms update rate, clients receive roughly 2 events/second.
- **Disconnect detection**: `request.is_disconnected()` is checked every 500ms. When the browser tab closes or navigates away, the generator exits cleanly.
- **`retry: 1000`**: Instructs the browser's `EventSource` to wait 1 second before reconnecting after a dropped connection. Without this, browsers may hammer the server immediately on reconnect.

**SSE event format (what the browser receives):**
```
retry: 1000

data: {"AAPL": {"ticker": "AAPL", "price": 191.23, "previous_price": 191.18,
       "timestamp": 1713614523.4, "change": 0.05, "change_percent": 0.0261,
       "direction": "up"}, "GOOGL": {...}, ...}

data: {"AAPL": {"ticker": "AAPL", "price": 191.19, ...}, ...}
```

---

### `market_data_demo.py` — Developer Dashboard

**What it does:** A standalone terminal application built with the `rich` library that demonstrates the market data system in action. Not part of the production server — it's a development and validation tool.

**Features:**
- Live table of all 10 tickers with price, change %, and sparkline mini-charts
- Color-coded direction (green up, red down)
- Event log showing shock events (>1% single-tick moves)
- 60-second session with summary statistics at end

**How to run:**
```bash
cd backend
uv run market_data_demo.py
```

**Why it exists:** Provides a quick visual sanity check of the simulator behavior without needing the full frontend. During development, it was used to validate that the GBM parameters, correlations, and shock events produced realistic-looking price action.

---

### `pyproject.toml` — Project Configuration

**What it does:** Defines the backend as a `uv` project with explicit dependencies and tool configuration.

**Runtime dependencies:**
| Package | Version | Purpose |
|---------|---------|---------|
| `fastapi` | ≥0.115.0 | Web framework, routing, request handling |
| `uvicorn[standard]` | ≥0.32.0 | ASGI server with WebSocket support |
| `numpy` | ≥2.0.0 | Cholesky decomposition, random sampling in GBM |
| `massive` | ≥1.0.0 | Polygon.io REST API client |
| `rich` | ≥13.0.0 | Terminal dashboard (demo only) |

**Dev dependencies:**
| Package | Purpose |
|---------|---------|
| `pytest` | Test runner |
| `pytest-asyncio` | Async test support |
| `pytest-cov` | Coverage reporting |
| `ruff` | Linting and formatting |

**`asyncio_mode = "auto"`**: All async test functions are automatically treated as async tests without needing `@pytest.mark.asyncio` on every test.

---

## How the Components Interact

### Data Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│  Environment: MASSIVE_API_KEY set?                               │
│  factory.create_market_data_source(cache)                        │
│                                                                  │
│  Yes ──▶ MassiveDataSource (Polygon.io REST, every 15s)         │
│  No  ──▶ SimulatorDataSource (GBMSimulator, every 500ms)        │
└──────────────────────────────────┬───────────────────────────────┘
                                   │  writes PriceUpdate objects
                                   ▼
┌──────────────────────────────────────────────────────────────────┐
│  PriceCache (thread-safe in-memory dict)                         │
│  {"AAPL": PriceUpdate(...), "GOOGL": PriceUpdate(...), ...}     │
│  version: 42187  (monotonic counter)                             │
└──────┬─────────────────────────────────┬────────────────────────┘
       │ reads (version-gated)            │ reads (on demand)
       ▼                                  ▼
┌─────────────────────┐      ┌─────────────────────────────────────┐
│  stream.py          │      │  Portfolio routes (planned)          │
│  GET /api/stream/   │      │  - /api/portfolio  (current prices) │
│  prices             │      │  - /api/portfolio/trade (fill price)│
│  (SSE)              │      │  Watchlist routes (planned)          │
│                     │      │  - /api/watchlist  (prices per tick)│
└────────┬────────────┘      └─────────────────────────────────────┘
         │  text/event-stream
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Browser (Frontend — Next.js / TypeScript)                       │
│                                                                  │
│  const es = new EventSource("/api/stream/prices");              │
│  es.onmessage = (e) => {                                        │
│    const prices = JSON.parse(e.data);   // {AAPL: {...}, ...}  │
│    updateWatchlist(prices);             // Flash green/red       │
│    updateSparklines(prices);            // Append to mini-chart │
│    updatePortfolioValue(prices);        // Recalculate total    │
│  };                                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Startup Sequence (Full Application)

When the FastAPI application boots (via uvicorn), the planned startup flow is:

1. **Database init** — Check if SQLite file exists; if not, create schema and seed default data (10 watchlist tickers, $10k cash balance, user profile).
2. **Create `PriceCache`** — Single instance shared across all routes and background tasks.
3. **Query watchlist from DB** — Get the user's current watchlist tickers.
4. **Create market data source** — `create_market_data_source(cache)` selects simulator or Massive based on env.
5. **Start the source** — `await source.start(tickers)` begins background price updates, seeds the cache immediately.
6. **Mount SSE router** — `app.include_router(create_stream_router(cache))` registers `/api/stream/prices`.
7. **Mount API routes** — Portfolio, watchlist, chat routes registered (planned).
8. **Serve static files** — Next.js build output served as static files for all non-`/api` paths.

### Watchlist Sync (Planned)

When the user adds or removes a ticker:
1. Frontend calls `POST /api/watchlist` or `DELETE /api/watchlist/{ticker}`
2. Backend route updates the SQLite `watchlist` table
3. Backend calls `await source.add_ticker(ticker)` or `await source.remove_ticker(ticker)`
4. The data source adds/removes the ticker from its active set
5. On next update cycle, the new ticker appears in the SSE stream
6. Frontend `EventSource` automatically picks it up and adds it to the watchlist panel

### Trade Execution (Planned)

When the user buys or sells shares:
1. Frontend calls `POST /api/portfolio/trade` with `{ticker, quantity, side}`
2. Backend reads current price from `cache.get_price(ticker)` — instant, no external call
3. Validates: enough cash for buys, enough shares for sells
4. Writes to SQLite: updates `positions`, deducts/adds `cash_balance`, appends to `trades`
5. Records a `portfolio_snapshot` immediately after the trade
6. Returns the executed trade with fill price

### AI Chat Flow (Planned)

When the user sends a chat message:
1. Backend loads portfolio context: reads positions from DB, calls `cache.get_all()` for live prices
2. Constructs prompt with portfolio state, conversation history, user message
3. Calls LiteLLM → OpenRouter (Cerebras provider) for fast inference
4. Parses structured JSON response: `{message, trades[], watchlist_changes[]}`
5. Auto-executes any trades via the same portfolio trade logic
6. Updates watchlist if requested
7. Stores message in `chat_messages` table
8. Returns full JSON response to frontend

---

## Backend ↔ Frontend Relationships

The backend and frontend communicate exclusively through the HTTP API. All requests go to the same origin (`localhost:8000`), so there are no CORS concerns.

### API Endpoints (Implemented and Planned)

| Endpoint | Method | Direction | Description |
|----------|--------|-----------|-------------|
| `/api/stream/prices` | GET | Server → Client (SSE) | Live price stream |
| `/api/portfolio` | GET | Client → Server | Current positions, cash, total value |
| `/api/portfolio/trade` | POST | Client → Server | Execute buy/sell |
| `/api/portfolio/history` | GET | Client → Server | P&L chart data points |
| `/api/watchlist` | GET | Client → Server | Watchlist with current prices |
| `/api/watchlist` | POST | Client → Server | Add ticker |
| `/api/watchlist/{ticker}` | DELETE | Client → Server | Remove ticker |
| `/api/chat` | POST | Client → Server | Send message, get AI response |
| `/api/health` | GET | Client → Server | Docker health check |

### Frontend Data Dependencies

Each frontend component and the backend data it needs:

| Frontend Component | Data Source |
|-------------------|-------------|
| **Watchlist panel** | SSE stream (`/api/stream/prices`) — live prices, flash animations |
| **Sparkline mini-charts** | SSE stream accumulated since page load — no backend storage |
| **Main chart area** | SSE stream accumulated for selected ticker |
| **Header (total value)** | SSE stream × position quantities (computed client-side) |
| **Portfolio heatmap** | `GET /api/portfolio` + SSE for live value updates |
| **Positions table** | `GET /api/portfolio` + SSE for live current price |
| **P&L chart** | `GET /api/portfolio/history` — backend snapshot history |
| **Trade bar** | `POST /api/portfolio/trade` — write-only, reads SSE for current price display |
| **AI chat panel** | `POST /api/chat` — request/response, no streaming |

### SSE as the Real-Time Backbone

The `EventSource` connection to `/api/stream/prices` is the heart of the frontend's real-time behavior. Every price-dependent component subscribes to the same stream:

```typescript
const eventSource = new EventSource("/api/stream/prices");

eventSource.onmessage = (event) => {
  const prices: Record<string, PriceUpdate> = JSON.parse(event.data);
  // Each component that cares about prices reads from this shared state
  updateWatchlistPrices(prices);
  flashPriceChanges(prices);         // Green/red CSS animation
  appendToSparklines(prices);        // Accumulate mini-chart points
  recomputePortfolioValue(prices);   // Total value in header
};
```

**Why accumulate sparklines client-side?** The backend doesn't store per-tick price history (only `portfolio_snapshots` every 30s). Sparklines show price action since page load — they're built up progressively in the browser from the SSE stream. This eliminates a storage/retrieval concern from the backend and makes the sparklines zero-cost to implement on the server side.

---

## Design Patterns Used

### Strategy Pattern — Market Data Sources
`MarketDataSource` ABC with two interchangeable implementations (`SimulatorDataSource`, `MassiveDataSource`). The rest of the system is unaware of which is active.

### Factory Pattern — Data Source Selection
`create_market_data_source(cache)` reads environment variables and returns the correct implementation. The caller just receives a `MarketDataSource` — an environment decision, not a code decision.

### Factory Pattern — SSE Router
`create_stream_router(cache)` injects the cache into the router closure, avoiding global variables while keeping FastAPI's router-based organization.

### Observer/Pub-Sub Pattern — SSE Streaming
The `PriceCache` is written to by one producer and read by N concurrent SSE connections (one per browser client). The version counter allows each SSE connection to track its own "last seen" state independently.

### Immutable Value Objects — `PriceUpdate`
`PriceUpdate` is frozen. Once created, it cannot change. Mutations produce new objects. This makes the cache safe to read without copying — the `PriceUpdate` you receive from `cache.get()` cannot be modified by the writer.

### Async/Threading Hybrid
FastAPI runs on asyncio. The GBM simulator is CPU-bound Python — safe to call from asyncio (fast, no I/O). The Massive REST client is blocking I/O — dispatched to a thread pool via `asyncio.to_thread()` to avoid blocking the event loop.

---

## Testing Architecture

### Test Organization

```
tests/
├── conftest.py                  # Minimal: event loop policy fixture
└── market/
    ├── test_models.py           # Unit: PriceUpdate math, serialization
    ├── test_cache.py            # Unit: PriceCache thread safety, versioning
    ├── test_simulator.py        # Unit: GBM math correctness, correlations
    ├── test_simulator_source.py # Integration: full source lifecycle
    ├── test_factory.py          # Unit: env-variable driven selection
    └── test_massive.py          # Unit: API client with mocked HTTP
```

**73 tests, 84% coverage.**

### Key Testing Approaches

**GBM validation (`test_simulator.py`):**
- Prices are always positive (GBM guarantee — confirmed by running many steps)
- Initial prices match seed values
- Pairwise correlation is tested statistically: run 1000 steps, compute Pearson correlation between ticker price series, verify tech-tech > 0.3, finance-finance > 0.2
- Adding/removing tickers rebuilds Cholesky without errors

**Thread-safety (`test_cache.py`):**
- Concurrent writes from multiple threads; verify no data corruption
- Version counter increments correctly under load

**Integration (`test_simulator_source.py`):**
- Full `start()` → `add_ticker()` → `remove_ticker()` → `stop()` lifecycle
- Cache is populated immediately after `start()`
- Errors in `step()` don't crash the loop

**API mocking (`test_massive.py`):**
- `massive.RESTClient` is mocked using `unittest.mock.patch`
- Tests verify correct cache updates, malformed snapshot handling, timestamp conversion

**Running tests:**
```bash
cd backend
uv sync --extra dev
uv run pytest -v                   # All tests with verbose output
uv run pytest --cov=app            # With coverage report
uv run pytest tests/market/test_cache.py  # Single module
```

---

## Environment Variables

| Variable | Required | Effect |
|----------|----------|--------|
| `MASSIVE_API_KEY` | No | If set, use Polygon.io for real market data. If absent/empty, use GBM simulator. |
| `OPENROUTER_API_KEY` | Yes (for chat) | LiteLLM → OpenRouter → Cerebras for AI chat responses. |
| `LLM_MOCK` | No | Set to `"true"` to return deterministic mock LLM responses (for E2E tests). |

---

## What Is Complete vs. Planned

### Complete

- `app/market/` — Full market data subsystem with both simulator and real-data implementations
- `GET /api/stream/prices` — SSE streaming endpoint
- 73 unit + integration tests with 84% coverage
- `market_data_demo.py` — Developer validation dashboard

### Planned (not yet implemented)

- `main.py` or `app/__init__.py` — FastAPI app creation, lifespan management, route mounting
- Database layer — SQLite schema, lazy initialization, seed data
- Portfolio routes — `/api/portfolio`, `/api/portfolio/trade`, `/api/portfolio/history`
- Watchlist routes — `/api/watchlist` CRUD
- Chat route — `/api/chat` with LiteLLM integration
- Portfolio snapshot background task
- Frontend — Next.js TypeScript static export
- Dockerfile — Multi-stage build (Node → Python)
- Start/stop scripts
- E2E tests (Playwright)
