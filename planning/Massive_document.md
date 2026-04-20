# Massive API Reference (formerly Polygon.io)

Research documentation for the Massive REST API — used in FinAlly to fetch real-time and end-of-day stock prices for multiple tickers.

---

## Overview

Polygon.io rebranded to **Massive** in October 2025. All existing API endpoints continue to work under both domains. The Python SDK package is now `massive`.

| Property | Value |
|----------|-------|
| Base URL | `https://api.polygon.io` (legacy) or `https://api.massive.com` |
| Auth method | API key — query param `apiKey=` or `Authorization: Bearer <key>` |
| Python package | `massive` (`uv add massive` / `pip install massive`) |
| Min Python | 3.9+ |
| Env var | `MASSIVE_API_KEY` |

---

## Authentication

**Query parameter** (simple, for testing):
```
GET https://api.polygon.io/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT&apiKey=YOUR_KEY
```

**Authorization header** (preferred in production):
```
Authorization: Bearer YOUR_KEY
```

**Python SDK** (handles auth automatically):
```python
from massive import RESTClient

client = RESTClient()                        # reads MASSIVE_API_KEY from env
client = RESTClient(api_key="your_key")      # or pass explicitly
```

---

## Rate Limits

| Tier | Price/mo | Calls/min | Data delay |
|------|----------|-----------|------------|
| Free | Free | 5 | 15-min delayed |
| Starter | $29 | 5 | 15-min delayed |
| Developer | $79 | 30–60 | Real-time |
| Professional | $199 | 120+ | Real-time |
| Enterprise | Custom | Unlimited | Real-time |

**FinAlly strategy**:
- Free tier → poll every **15 seconds** (stay under 5 req/min)
- Paid tier → poll every **2–5 seconds**

---

## Key Endpoints

### 1. Multi-Ticker Snapshot (Primary)

Fetches current price data for up to **250 tickers in a single API call**. This is the correct endpoint for a watchlist-style application.

**REST endpoint**:
```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT
```

**Python SDK**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price:.2f}")
    print(f"  Change: {snap.day.change_percent:+.2f}%")
    print(f"  Day range: ${snap.day.low:.2f} – ${snap.day.high:.2f}")
    print(f"  Volume: {snap.day.volume:,}")
```

**Full response schema** (per ticker):
```json
{
  "ticker": "AAPL",
  "day": {
    "open": 129.61,
    "high": 130.15,
    "low": 125.07,
    "close": 125.07,
    "volume": 111237700,
    "volume_weighted_average_price": 127.35,
    "previous_close": 129.61,
    "change": -4.54,
    "change_percent": -3.50
  },
  "last_trade": {
    "price": 125.07,
    "size": 100,
    "exchange": "XNYS",
    "timestamp": 1675190399000
  },
  "last_quote": {
    "bid_price": 125.06,
    "ask_price": 125.08,
    "bid_size": 500,
    "ask_size": 1000,
    "spread": 0.02,
    "timestamp": 1675190399500
  },
  "prev_daily_bar": {
    "open": 132.00,
    "high": 133.50,
    "low": 129.00,
    "close": 129.61,
    "volume": 98000000
  },
  "min": {
    "open": 125.00,
    "high": 125.20,
    "low": 124.90,
    "close": 125.07,
    "volume": 450000
  }
}
```

**Fields we use in FinAlly**:
| Field | Purpose |
|-------|---------|
| `last_trade.price` | Current price — used for trades and display |
| `last_trade.timestamp` | When the price was recorded (Unix ms) |
| `day.previous_close` | Prior close — for day change % calculation |
| `day.change_percent` | Day change percentage display |
| `day.open`, `day.high`, `day.low`, `day.close` | OHLC for chart tooltip |
| `day.volume` | Volume display |

---

### 2. Single Ticker Snapshot

Detailed data for one ticker. Used when a user clicks a ticker for the detail view.

**REST endpoint**:
```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{ticker}
```

**Python SDK**:
```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)

print(f"Price:     ${snapshot.last_trade.price:.2f}")
print(f"Bid/Ask:   ${snapshot.last_quote.bid_price:.2f} / ${snapshot.last_quote.ask_price:.2f}")
print(f"Day range: ${snapshot.day.low:.2f} – ${snapshot.day.high:.2f}")
print(f"Volume:    {snapshot.day.volume:,}")
```

---

### 3. Previous Close (Single Ticker)

Previous day's OHLC bar. Useful for seeding the simulator with realistic starting prices, or for displaying reference prices during pre-market hours.

**REST endpoint**:
```
GET /v2/aggs/ticker/{ticker}/prev?adjusted=true
```

**Python SDK**:
```python
prev = client.get_previous_close_agg(ticker="AAPL")

for agg in prev:
    print(f"Previous close: ${agg.close:.2f}")
    print(f"OHLC: O={agg.open} H={agg.high} L={agg.low} C={agg.close}")
    print(f"Volume: {agg.volume:,}")
```

**Response**:
```json
{
  "ticker": "AAPL",
  "results": [
    {
      "o": 150.00,
      "h": 155.00,
      "l": 149.00,
      "c": 154.50,
      "v": 1000000,
      "vw": 152.35,
      "t": 1672531200000,
      "n": 320000
    }
  ],
  "status": "OK",
  "count": 1
}
```

Field aliases: `o`=open, `h`=high, `l`=low, `c`=close, `v`=volume, `vw`=VWAP, `t`=timestamp (ms), `n`=trade count.

---

### 4. Daily Open-Close (Historical EOD)

Full OHLC for a specific date. For historical chart data or backlooking analysis.

**REST endpoint**:
```
GET /v1/open-close/{ticker}/{date}?adjusted=true
```

**Example**:
```
GET /v1/open-close/AAPL/2024-01-15?adjusted=true&apiKey=YOUR_KEY
```

**Response**:
```json
{
  "status": "OK",
  "from": "2024-01-15",
  "symbol": "AAPL",
  "open": 185.94,
  "high": 186.22,
  "low": 183.43,
  "close": 183.63,
  "volume": 59148247,
  "afterHours": 183.75,
  "preMarket": 185.10,
  "otc": false
}
```

---

### 5. Aggregate Bars (Historical OHLCV)

Time-series of OHLCV bars over a date range. Used for rendering historical price charts.

**REST endpoint**:
```
GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}
```

**Python SDK**:
```python
aggs = list(client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2024-01-01",
    to="2024-01-31",
    adjusted=True,
    limit=50000,
))

for a in aggs:
    dt = datetime.fromtimestamp(a.timestamp / 1000)
    print(f"{dt.date()}: O={a.open:.2f} H={a.high:.2f} L={a.low:.2f} C={a.close:.2f} V={a.volume:,}")
```

**Valid timespans**: `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`

**Response** (per bar):
```json
{
  "o": 130.00,
  "h": 132.50,
  "l": 129.80,
  "c": 131.20,
  "v": 50000000,
  "vw": 130.95,
  "t": 1672531200000,
  "n": 410000
}
```

---

### 6. Last Trade / Last Quote

Convenience endpoints for the most recent trade or NBBO quote for a single ticker.

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price:.2f} × {trade.size} shares")

quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid:.2f} × {quote.bid_size}")
print(f"Ask: ${quote.ask:.2f} × {quote.ask_size}")
```

---

### 7. Market Status

Whether the market is currently open, closed, or in extended hours.

**REST endpoint**:
```
GET /v1/marketstatus/now
```

**Response**:
```json
{
  "market": "open",
  "serverTime": "2024-01-15T14:30:00.000Z",
  "exchanges": {
    "nasdaq": "open",
    "nyse": "open",
    "otc": "open"
  },
  "currencies": {
    "fx": "open",
    "crypto": "open"
  }
}
```

**Python SDK**:
```python
status = client.get_market_status()
is_open = status.market == "open"
```

Market status values: `open`, `closed`, `early_trading`, `late_trading`

---

## Error Handling

The Python client raises exceptions for HTTP errors. Common codes:

| Status | Meaning | Action |
|--------|---------|--------|
| `401` | Invalid API key | Check `MASSIVE_API_KEY` env var |
| `403` | Plan doesn't include endpoint | Upgrade tier or use different endpoint |
| `429` | Rate limit exceeded | Back off; free tier is 5 req/min |
| `5xx` | Server error | Client retries 3× automatically |

```python
from massive.exceptions import BadResponse

try:
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
except BadResponse as e:
    if e.status == 429:
        # Rate limited — increase poll interval
        logger.warning("Rate limited by Massive API, backing off")
    elif e.status == 401:
        logger.error("Invalid MASSIVE_API_KEY")
    else:
        logger.error(f"Massive API error {e.status}: {e}")
```

---

## Behavior Notes

- **Multi-ticker snapshot is one API call** — critical for staying within free tier limits
- **Timestamps** are Unix milliseconds; divide by 1000 for Unix seconds
- **After market close**, `last_trade.price` reflects the last traded price (may include after-hours trades)
- **`day` object** resets at market open; pre-market values may reflect the previous session
- **`previous_close`** in the `day` object is the prior day's closing price — use this for day-change calculations
- **Pagination**: snapshot endpoint returns up to 250 tickers; for watchlists >250 tickers, split into batches
