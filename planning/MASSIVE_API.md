# Massive API Reference (formerly Polygon.io)

Verified reference documentation for the Massive REST API and its official Python
client, as used by FinAlly's `MassiveDataSource` (`backend/app/market/massive_client.py`).

Researched directly against the vendor's current docs and the `massive-com/client-python`
source on 2026-09-04 (sources listed at the bottom). This supersedes the original
pre-verification draft of this document, which got several Python attribute names
wrong (see "Corrections vs. the Earlier Draft" below).

## Overview

- **Company**: Polygon.io rebranded to **Massive** on **October 30, 2025**. Existing
  API keys, accounts, and billing carried over unchanged.
- **Base URL**: `https://api.massive.com` (new default). The legacy
  `https://api.polygon.io` host still works and will continue to for an extended
  period, so old integrations aren't broken by the rename.
- **Python package**: `massive` on PyPI — install with `uv add massive` or
  `pip install -U massive`. It is a fork/continuation of the old `polygon-api-client`
  package, republished under the new name with the new default base URL.
- **Min Python version**: 3.9+
- **Repo**: [github.com/massive-com/client-python](https://github.com/massive-com/client-python)

## Authentication

```python
from massive import RESTClient

# No-arg form: reads the MASSIVE_API_KEY environment variable automatically.
client = RESTClient()

# Or pass explicitly:
client = RESTClient(api_key="your_key_here")
```

`MASSIVE_API_KEY` is the client library's own default environment variable name —
it isn't something FinAlly invented to match; the project's `.env` variable and the
client's built-in default happen to line up, so `RESTClient()` with no arguments
just works once the process environment has `MASSIVE_API_KEY` set.

Internally, every request carries `Authorization: Bearer <API_KEY>`; the client sets
this header for you.

## Rate Limits

| Tier | Limit |
|------|-------|
| Free | 5 requests/minute |
| Paid (all tiers) | No hard cap; vendor asks that you stay under ~100 req/s so you don't degrade service for others |

FinAlly polls on a timer rather than opening a persistent connection (see "Why
polling, not WebSockets" below). `MassiveDataSource` defaults to a 15s interval,
which keeps a single-user instance comfortably under the free-tier 5/min limit
even with retries. Paid tiers can safely poll every 2-5s.

## Client Initialization Details

- `RESTClient(api_key=..., connect_timeout=..., read_timeout=..., retries=...)` —
  `retries` and timeouts are optional overrides; the client has built-in retry with
  exponential backoff (backoff factor 0.1s) on `413, 429, 499, 500, 502, 503, 504`.
- `RESTClient(pagination=False)` disables automatic multi-page fetching for
  paginated list methods (`list_aggs`, `list_trades`, `list_quotes`, etc.); with
  pagination on (the default), `limit` controls *page size*, and the client
  transparently walks all pages for you.
- `RESTClient(trace=True, verbose=True)` prints request/response details — useful
  for debugging response shapes during development.

## Endpoint FinAlly Actually Uses

### Snapshot — All Tickers (the only endpoint the poller calls)

This is the sole call `MassiveDataSource._fetch_snapshots()` makes. It returns
current prices for an arbitrary list of tickers in **one HTTP round trip**, which
is what makes REST polling viable within the free-tier rate limit.

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python method** (`SnapshotClient.get_snapshot_all`):
```python
def get_snapshot_all(
    self,
    market_type: str | SnapshotMarketType,
    tickers: str | list[str] | None = None,
    include_otc: bool | None = False,
    params: dict | None = None,
    raw: bool = False,
    options: RequestOptionBuilder | None = None,
) -> list[TickerSnapshot] | HTTPResponse
```

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
```

**`TickerSnapshot` fields** (Python attribute names on the typed model — see
"Raw JSON vs. Python attributes" below for how these map to the wire format):

| Attribute | Type | Meaning |
|---|---|---|
| `ticker` | `str` | Symbol |
| `day` | `Agg` | Current session's running bar (open/high/low/close/volume/vwap) |
| `prev_day` | `Agg` | Previous session's full bar — this is where "previous close" actually lives (`prev_day.close`), **not** `day.previous_close` |
| `min` | `MinuteSnapshot` | Most recently completed minute bar |
| `last_trade` | `LastTrade` | Most recent trade: `price`, `size`, `exchange`, `sip_timestamp` (Unix **nanoseconds**), etc. |
| `last_quote` | `LastQuote` | Most recent NBBO quote: `bid_price`, `ask_price`, `bid_size`, `ask_size`, `sip_timestamp`, etc. |
| `todays_change` | `float` | Absolute change since previous close |
| `todays_change_percent` | `float` | Percent change since previous close |
| `updated` | `int` | Server-side update timestamp |
| `fair_market_value` | `float \| None` | Business-plan-only field; `None` otherwise |

`massive_client.py` only reads `snap.ticker`, `snap.last_trade.price`, and
`snap.last_trade.timestamp` — everything else above is available but unused today.
`todays_change_percent` is a ready-made source for a "daily change %" column if the
watchlist UI wants one without computing it client-side.

⚠️ **Timestamp units differ by object.** `last_trade.sip_timestamp` /
`last_trade.timestamp` (the attribute the client normalizes it to) is Unix
**milliseconds**, matching what `massive_client.py` assumes (`timestamp / 1000.0`).
Some other Massive timestamp fields are nanoseconds — always check the specific
model before assuming a unit.

## Endpoints Available But Not Used Today

Documented here because they're natural next steps (e.g., for historical chart
backfill) and because the earlier draft of this document described some of them
with the wrong field names.

### Previous Close

**REST**: `GET /v2/aggs/ticker/{ticker}/prev`

```python
prev = client.get_previous_close_agg(ticker="AAPL")  # -> PreviousCloseAgg
print(prev.close, prev.open, prev.high, prev.low, prev.volume, prev.timestamp)
```

`PreviousCloseAgg` fields: `ticker`, `open`, `high`, `low`, `close`, `volume`,
`vwap`, `timestamp`. Redundant with `TickerSnapshot.prev_day` if you're already
calling `get_snapshot_all`; useful standalone if you only need yesterday's close
without pulling a full snapshot.

### Aggregates / Bars (for historical chart data)

**REST**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

```python
for bar in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2026-08-01",
    to="2026-09-01",
    adjusted=True,
    sort="asc",
    limit=50000,
):
    print(bar.timestamp, bar.open, bar.high, bar.low, bar.close, bar.volume)
```

`list_aggs` auto-paginates (returns an `Iterator[Agg]`); `get_aggs` is the
non-paginating sibling that returns a plain `list[Agg]`. `Agg` fields: `open`,
`high`, `low`, `close`, `volume`, `vwap`, `timestamp`, `transactions`, `otc`.
This is the endpoint to reach for if the main chart ever needs to seed itself with
real historical bars instead of only the live SSE stream accumulated since page load.

### Last Trade / Last Quote (single ticker, no snapshot)

```python
trade = client.get_last_trade(ticker="AAPL")   # -> LastTrade
quote = client.get_last_quote(ticker="AAPL")   # -> LastQuote
```

Only useful if you want just one ticker's trade or quote without the rest of a
snapshot; for FinAlly's "poll the whole watchlist" pattern, `get_snapshot_all` is
strictly better since it's one call instead of N.

### Universal Snapshot (newer, cross-asset-class endpoint)

`list_universal_snapshots(type=SnapshotMarketType.STOCKS, ticker_any_of=[...])`
is a newer, more general snapshot endpoint that also covers options/indices/forex/crypto
in one shape. It caps at **250 symbols per request** (vs. no documented cap on
`get_snapshot_all`'s `tickers` param). `get_snapshot_all` is not deprecated and
remains the simpler choice for FinAlly's all-stocks use case; `list_universal_snapshots`
would only matter if the watchlist needed to mix asset classes.

## Raw JSON vs. Python Attributes

The vendor's raw JSON uses `camelCase` (`lastTrade`, `prevDay`, `lastQuote`,
`todaysChangePerc`), but the official Python client deserializes responses into
typed dataclasses with `snake_case` attributes (`last_trade`, `prev_day`,
`last_quote`, `todays_change_percent`). **Code should always use the Python
attribute names**, not the raw JSON field names — mixing the two up is the most
common way to introduce a silent `AttributeError` or, worse, a `None` read that
looks like a valid price. Pass `raw=True` to any client method to get the
untouched JSON/`HTTPResponse` instead, if you ever need to inspect the wire format
directly.

## WebSocket Client (available, not used by FinAlly)

```python
from massive import WebSocketClient

ws = WebSocketClient(api_key="...", subscriptions=["T.AAPL"])  # T. = trades
ws.run(handle_msg=lambda msgs: print(msgs))
```

FinAlly deliberately polls REST instead of using this (see PLAN.md §6): a
WebSocket connection is stateful, requires reconnect/backoff logic, and needs a
paid plan for real-time trade/quote streams (delayed data is more limited on
WebSocket than on the free-tier snapshot REST endpoint). Polling is simpler,
works identically across all plan tiers, and the snapshot endpoint's "all tickers
in one call" shape maps cleanly onto FinAlly's shared `PriceCache` model.

## Error Handling

The client raises typed exceptions from `massive.exceptions`:

- **`AuthError`** — empty or invalid API key (HTTP 401/403 conditions)
- **`BadResponse`** — any non-2xx response the retry policy didn't recover from

In practice, expect:
- **401** — invalid API key
- **403** — plan doesn't include the endpoint/data you requested
- **429** — rate limit exceeded (free tier: 5 req/min) — the client retries these
  automatically per the backoff policy above before raising
- **5xx** — server errors — also auto-retried

`massive_client.py`'s `_poll_once()` wraps the whole poll in a broad
`try/except Exception`, logs, and lets the next scheduled poll retry — it doesn't
distinguish `AuthError` from `BadResponse` today. A bad API key currently just
produces a silent, permanently-empty price cache with an error logged every
15 seconds rather than a startup-time failure; if that's ever worth surfacing to
the user, catching `AuthError` specifically at startup (during the immediate
first poll in `start()`) and failing fast is the natural place to do it.

## Notes

- The snapshot endpoint returns data for **all requested tickers in one call** —
  critical for staying within the free-tier rate limit.
- During market-closed hours, `last_trade.price` reflects the last traded price
  (may include after-hours activity).
- The `day` bar resets at market open; during pre-market it may still reflect the
  previous session. Use `prev_day` when you specifically want "yesterday's close,"
  never `day`.
- FinAlly's dynamic-ticker behavior (PLAN.md §6) falls directly out of this
  endpoint's shape: an unrecognized/invalid symbol simply doesn't appear in the
  `tickers` list of the response — there's no explicit "invalid ticker" error, so
  "does the price cache have this ticker" is the only reliable validity check.

## Corrections vs. the Earlier Draft

The original draft of this document (written before this verification pass) had
two inaccuracies worth flagging so they aren't propagated:

1. It read previous close as `snap.day.previous_close` — that field doesn't
   exist. Previous close is `snap.prev_day.close`.
2. It read day change as `snap.day.change_percent` — that field lives at the
   top level as `snap.todays_change_percent`, not nested under `day`.

Neither bug was ever reachable in production: `massive_client.py` only ever
reads `last_trade.price` and `last_trade.timestamp`, so the wrong field names in
the earlier draft were never actually executed.

## Sources

- [massive-com/client-python README](https://github.com/massive-com/client-python/blob/master/README.md)
- [massive-com/client-python repository](https://github.com/massive-com/client-python) (`massive/rest/snapshot.py`, `massive/rest/aggs.py`, `massive/rest/trades.py`, `massive/rest/quotes.py`, `massive/rest/models/snapshot.py`, `massive/rest/models/aggs.py`, `massive/rest/models/trades.py`, `massive/rest/models/quotes.py`, `massive/rest/base.py`, `massive/exceptions.py`)
- [Polygon.io is Now Massive](https://massive.com/blog/polygon-is-now-massive)
- [Full Market Snapshot | Stocks REST API](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [What is the max number of tickers I can pass through Massive's Snapshot?](https://massive.com/knowledge-base/article/what-is-the-max-number-of-tickers-i-can-pass-through-massives-snapshot)
- [What is the request limit for Massive's RESTful APIs?](https://massive.com/knowledge-base/article/what-is-the-request-limit-for-massives-restful-apis)
- [Custom Bars | Stocks REST API](https://massive.com/docs/rest/stocks/aggregates/custom-bars)
