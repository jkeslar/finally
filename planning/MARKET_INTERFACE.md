# Market Data Interface Design

Unified Python interface for market data in FinAlly. Two implementations — the GBM
simulator and the Massive API poller — sit behind one abstract interface, so all
downstream code (SSE streaming, trade validation, the frontend) is agnostic to
where prices actually come from.

**Status**: this describes the interface as actually implemented in
`backend/app/market/` (see `planning/MARKET_DATA_SUMMARY.md` for the build
summary). It supersedes the original pre-implementation sketch of this document;
a few details below (the data model's shape, how the cache computes `change`,
factory logging) changed slightly during implementation. For the Massive-specific
half of this, see `planning/MASSIVE_API.md`; for the simulator half, see
`planning/MARKET_SIMULATOR.md`.

## Core Data Model

`app/market/models.py`:

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float: ...          # price - previous_price, rounded to 4dp
    @property
    def change_percent(self) -> float: ...  # % change, rounded to 4dp; 0.0 if previous_price == 0
    @property
    def direction(self) -> str: ...          # "up" | "down" | "flat"

    def to_dict(self) -> dict: ...           # JSON-serializable form for SSE
```

`PriceUpdate` is the only structure that leaves the market data layer. `change`,
`change_percent`, and `direction` are computed **properties**, not stored fields —
they're derived once from `price`/`previous_price` at read time rather than
recomputed and frozen at write time. This is a deliberate simplification over the
original design sketch (which computed and stored `change`/`direction` inside
`PriceCache.update()`): keeping `PriceUpdate` a two-price, one-timestamp record
means there's a single source of truth for the derived fields no matter who
constructs the object.

## Abstract Interface

`app/market/interface.py`:

```python
class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None: ...
    @abstractmethod
    async def stop(self) -> None: ...
    @abstractmethod
    async def add_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None: ...
    @abstractmethod
    def get_tickers(self) -> list[str]: ...
```

Both `SimulatorDataSource` and `MassiveDataSource` implement this. Neither
`start()` nor the periodic update loop returns prices to the caller — they write
into a shared `PriceCache`, which is the only thing readers ever touch. `start()`
must be called exactly once (calling it twice is undefined); `stop()` is safe to
call multiple times, matching FastAPI's lifespan shutdown semantics where cleanup
code can run more than once in edge cases.

## Price Cache

`app/market/cache.py` — the single point of truth both sources write to and every
reader (SSE stream, portfolio valuation, trade execution) reads from:

```python
class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker."""

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate: ...
    def get(self, ticker: str) -> PriceUpdate | None: ...
    def get_all(self) -> dict[str, PriceUpdate]: ...
    def get_price(self, ticker: str) -> float | None:   # convenience: just the float
    def remove(self, ticker: str) -> None: ...

    @property
    def version(self) -> int: ...   # monotonic counter, bumped on every update()
```

Key implementation details:

- **Thread-safe via a plain `threading.Lock`**, not an asyncio lock — both
  `SimulatorDataSource` (asyncio task) and `MassiveDataSource` (asyncio task using
  `asyncio.to_thread` for the blocking HTTP call) write from the event loop, so a
  standard lock is sufficient; there's no true multi-threaded contention today,
  but it costs nothing and protects against future multi-worker deployment.
- **Rounds to 2 decimal places on write** (`round(price, 2)`), so every reader —
  SSE payloads, trade fills, portfolio valuation — sees already-display-clean
  prices without each having to round independently.
- **`version` is the SSE change-detection mechanism**: bumped on every `update()`
  call, so the stream endpoint (see below) can cheaply check "has anything changed
  since I last sent a payload" without diffing the whole price dict.
- On the very first `update()` for a ticker, `previous_price` is seeded equal to
  `price` (no prior value exists), so `direction` correctly reads `"flat"` and
  `change` is `0.0` instead of raising or defaulting to some sentinel.

## Factory Function

`app/market/factory.py` — selects the implementation at startup:

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """MASSIVE_API_KEY set and non-empty -> MassiveDataSource; otherwise -> SimulatorDataSource."""
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

`.strip()` matters: an `.env` file with `MASSIVE_API_KEY=` (present but empty, or
whitespace) must fall through to the simulator rather than instantiating a
`MassiveDataSource` with a blank key that would fail every poll with `AuthError`.
The factory returns an **unstarted** source — the caller (FastAPI's lifespan
handler) still owns calling `await source.start(initial_tickers)`.

## The Two Implementations

### `MassiveDataSource` (`massive_client.py`)

Polls `get_snapshot_all()` for the full watchlist on a timer (default 15s — see
`planning/MASSIVE_API.md` for rate-limit rationale). Notable behavior:

- **Runs an immediate poll inside `start()`**, before the periodic task even
  begins, so the cache has real data as soon as the app comes up instead of
  waiting a full interval for the first prices to appear.
- **Offloads the client call with `asyncio.to_thread`** — the `massive` package's
  `RESTClient` is synchronous, so calling it directly from `_poll_once()` would
  block the whole event loop (including the SSE stream and every other request)
  for the duration of the HTTP round trip.
- **Ticker add/remove are cheap, in-memory list edits** — `add_ticker()` just
  appends to `self._tickers`; the new symbol shows up automatically on the next
  scheduled poll rather than triggering an immediate extra API call. `remove_ticker()`
  additionally calls `self._cache.remove(ticker)` so a delisted-from-watchlist
  symbol disappears from the SSE stream immediately rather than lingering with a
  stale last-known price until it ages out.
- **Poll failures are caught, logged, and swallowed** — a single bad poll (rate
  limit, transient network error) doesn't crash the background task; the loop
  simply tries again on the next interval. See `planning/MASSIVE_API.md`'s "Error
  Handling" section for the gap this leaves (an invalid API key fails silently
  forever rather than at startup).

### `SimulatorDataSource` (`simulator.py`)

Wraps a `GBMSimulator` (see `planning/MARKET_SIMULATOR.md`) in the same async-task
shape, at a 0.5s default interval instead of 15s:

- **Seeds the cache synchronously inside `start()`** (and inside `add_ticker()`)
  by reading `self._sim.get_price(ticker)` immediately after adding it — so a
  newly-added ticker has a price in the cache before the next 0.5s tick, matching
  the Massive side's "don't make the caller wait a full interval" behavior.
- **The per-tick loop wraps `self._sim.step()` in `try/except Exception`** and
  logs on failure without stopping the loop — a single bad step (e.g. a NaN from
  a pathological correlation matrix) shouldn't take down price streaming
  entirely.

## Integration with SSE

`app/market/stream.py`'s `GET /api/stream/prices` endpoint reads `PriceCache`
directly — it has no knowledge of which `MarketDataSource` is active:

```python
current_version = price_cache.version
if current_version != last_version:
    last_version = current_version
    prices = price_cache.get_all()
    if prices:
        yield f"data: {json.dumps({t: u.to_dict() for t, u in prices.items()})}\n\n"
await asyncio.sleep(interval)   # 0.5s poll-the-cache loop, independent of source cadence
```

Using `version` instead of a naive "send every 0.5s regardless" means the stream
only emits a payload when something actually changed, and it degrades gracefully
across sources: the 15s-cadence Massive source and the 0.5s-cadence simulator both
"just work" against the same 0.5s cache-polling loop — the SSE endpoint simply
sends less often when the underlying source is slower. The endpoint also yields a
`retry: 1000\n\n` directive up front so `EventSource`'s built-in reconnect logic
retries quickly after a drop, and it exits its loop via `request.is_disconnected()`
so a closed browser tab doesn't leave an orphaned generator running forever.

## Trade & Watchlist Validation

Per PLAN.md §6 and §8, "is this ticker tradable" is answered by one check —
**does `PriceCache` have a current price for it** — never a separate hardcoded
symbol list:

```python
if price_cache.get_price(ticker) is None:
    raise ValidationError(f"{ticker} has no current price")
```

This is what makes the two sources' very different "unknown ticker" behaviors
converge on identical downstream behavior: the simulator invents a price for any
ticker (see `planning/MARKET_SIMULATOR.md`), so it always passes; Massive simply
never returns a snapshot for an invalid symbol, so it's absent from the cache and
naturally fails the same check with no special-case code required.

## File Structure (as built)

```
backend/
  app/
    market/
      __init__.py          # re-exports the public surface (see backend/CLAUDE.md)
      models.py             # PriceUpdate
      interface.py           # MarketDataSource ABC
      cache.py               # PriceCache
      factory.py              # create_market_data_source()
      massive_client.py       # MassiveDataSource
      simulator.py             # GBMSimulator + SimulatorDataSource
      seed_prices.py            # SEED_PRICES, TICKER_PARAMS, correlation constants
      stream.py                  # create_stream_router() — SSE endpoint factory
  tests/
    market/                       # 73 tests, 84% coverage — see MARKET_DATA_SUMMARY.md
```

## Lifecycle

1. **App startup** (FastAPI lifespan): create `PriceCache()`, call
   `create_market_data_source(cache)`, then `await source.start(initial_tickers)`.
2. **Watchlist changes**: the watchlist API route calls
   `await source.add_ticker(ticker)` / `await source.remove_ticker(ticker)` — the
   route layer never touches `PriceCache` directly for adds, only the source does
   (removal explicitly clears the cache entry too, per above).
3. **SSE streaming**: `GET /api/stream/prices` reads `PriceCache.get_all()` on its
   own 0.5s cadence, independent of either source's update interval.
4. **Trade execution**: reads the current price via `PriceCache.get_price(ticker)`
   at fill time — never caches or re-reads a stale price within a single request.
5. **App shutdown**: `await source.stop()`, cancelling the background task
   cleanly.

## Public Import Surface

Per `backend/CLAUDE.md`, downstream code imports from the package root, not the
individual modules:

```python
from app.market import PriceCache, PriceUpdate, MarketDataSource, create_market_data_source, create_stream_router
```
