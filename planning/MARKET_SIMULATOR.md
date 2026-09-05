# Market Simulator Design

Approach and code structure for simulating realistic stock prices when no
`MASSIVE_API_KEY` is configured. This is the default data source — most students
running FinAlly for the first time never touch the Massive API at all.

**Status**: this describes the simulator as actually implemented in
`backend/app/market/simulator.py` and `backend/app/market/seed_prices.py`,
covered by 17 tests in `backend/tests/market/test_simulator.py` plus 10
integration tests in `test_simulator_source.py` (98% line coverage on
`simulator.py`). It supersedes the original pre-implementation sketch of this
document; the math and structure below match what shipped almost exactly, with
the differences noted inline. For where this plugs into the rest of the market
data layer, see `planning/MARKET_INTERFACE.md`.

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** — the standard model
underlying Black-Scholes option pricing — to generate price paths that evolve
continuously with random noise, can never go negative, and produce the lognormal
return distribution seen in real markets. `SimulatorDataSource` steps the
simulation every 500ms via an asyncio background task, producing a continuous
stream of small, plausible-looking price changes.

## GBM Math

At each time step:

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

- `S(t)` — current price
- `mu` — annualized drift (expected return), e.g. `0.05` for 5%/year
- `sigma` — annualized volatility, e.g. `0.20` for 20%/year
- `dt` — this time step, expressed as a fraction of a trading year
- `Z` — a (correlated) standard normal draw

`dt` is derived from real trading-calendar constants rather than a round number,
so the annualized `mu`/`sigma` parameters translate to realistic per-tick moves:

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600   # 5,896,800 (252 trading days * 6.5h * 3600s)
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR    # ~8.48e-8, for 500ms ticks
```

This tiny `dt` produces sub-cent moves per individual tick, which accumulate
naturally into realistic-looking intraday ranges over the course of a session —
e.g. TSLA's `sigma=0.50` produces roughly the right order of magnitude of
intraday range over a full simulated trading day.

## Correlated Moves

Real stocks don't move independently — tech names tend to move together on
market-wide news, etc. The simulator generates correlated random draws via a
**Cholesky decomposition** of a correlation matrix: given correlation matrix `C`,
compute `L = cholesky(C)`, then for independent standard normals `z_independent`,
`z_correlated = L @ z_independent` has the desired covariance structure.

Correlation groups and coefficients (`seed_prices.py`):

```python
CORRELATION_GROUPS = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}
INTRA_TECH_CORR = 0.6      # tech stocks move together
INTRA_FINANCE_CORR = 0.5   # finance stocks move together
CROSS_GROUP_CORR = 0.3     # different sectors, or an unrecognized ticker
TSLA_CORR = 0.3            # TSLA does its own thing, even though it's in the tech set
```

The pairwise lookup (`GBMSimulator._pairwise_correlation`) checks TSLA **first**,
before the sector-membership checks — TSLA is a member of the `"tech"` set (so
that unknown-ticker fallback logic elsewhere doesn't need a special case for it),
but it's deliberately given the flat `0.3` cross-group correlation with
*everything*, including other tech names, rather than the `0.6` intra-tech rate.
A ticker outside both named groups (any dynamically-added symbol) also falls
through to `0.3` — one constant serves double duty as both "cross-sector" and
"unknown ticker" correlation, since both cases mean "no special relationship
assumed."

The Cholesky matrix is rebuilt (`_rebuild_cholesky()`) on every `add_ticker()`/
`remove_ticker()` call — O(n²) matrix construction plus an O(n³) decomposition,
but `n` stays well under 50 tickers in practice, so this is not a performance
concern. With 0 or 1 tickers, `_cholesky` is left `None` and `step()` uses the
independent draws directly (a 1x1 correlation matrix is trivially just itself,
so skipping the decomposition in that case is a harmless shortcut, not an
approximation).

## Random Events

Each tick, each ticker independently has a small chance of a sudden 2-5% shock,
for visual drama on the dashboard:

```python
if random.random() < event_probability:   # default 0.001 (0.1%)
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

At 2 ticks/second, 0.1% per tick per ticker works out to roughly one event per
ticker every ~500 seconds; across a 10-ticker default watchlist, expect a
noticeable jump somewhere on the board roughly every ~50 seconds — frequent
enough to keep a live demo visually interesting without every ticker looking
like it's constantly spiking.

## Seed Prices & Per-Ticker Parameters

`seed_prices.py` holds realistic starting prices and volatility/drift parameters
for the ten default watchlist tickers:

```python
SEED_PRICES = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00, "TSLA": 250.00,
    "NVDA": 800.00, "META": 500.00, "JPM": 195.00, "V": 280.00, "NFLX": 600.00,
}

TICKER_PARAMS = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # high volatility
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # high volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}   # for any ticker not in the table above
```

## Dynamically Added Tickers

Per PLAN.md §6, a ticker the user (or the AI chat) adds beyond the ten defaults
still needs to "just work" in simulator mode — there's no such thing as an
invalid symbol to the simulator:

```python
def _add_ticker_internal(self, ticker: str) -> None:
    if ticker in self._prices:
        return
    self._tickers.append(ticker)
    self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
    self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))
```

An unrecognized ticker gets a random starting price in `$50–$300` and the
`DEFAULT_PARAMS` volatility/drift (`sigma=0.25, mu=0.05` — a middling profile,
neither as sleepy as JPM/V nor as wild as TSLA/NVDA), and its correlation with
every other ticker falls back to `CROSS_GROUP_CORR` (`0.3`) since it's in neither
named sector set. It has a live, moving price from the very next tick — there is
no "unknown ticker" error path in the simulator at all, by design.

## Implementation Structure

`GBMSimulator` is the pure simulation engine — no asyncio, no cache access, just
ticker state and a `step()` method:

```python
class GBMSimulator:
    def step(self) -> dict[str, float]:
        """Advance all tickers by one time step. Returns {ticker: new_price}."""
        n = len(self._tickers)
        if n == 0:
            return {}
        z_independent = np.random.standard_normal(n)
        z_correlated = self._cholesky @ z_independent if self._cholesky is not None else z_independent
        result = {}
        for i, ticker in enumerate(self._tickers):
            mu, sigma = self._params[ticker]["mu"], self._params[ticker]["sigma"]
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_correlated[i]
            self._prices[ticker] *= math.exp(drift + diffusion)
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock
            result[ticker] = round(self._prices[ticker], 2)
        return result

    def add_ticker(self, ticker: str) -> None: ...      # adds + rebuilds Cholesky
    def remove_ticker(self, ticker: str) -> None: ...   # removes + rebuilds Cholesky
    def get_price(self, ticker: str) -> float | None: ...
    def get_tickers(self) -> list[str]: ...
```

`SimulatorDataSource` is the thin `MarketDataSource` adapter around it — it owns
the asyncio task, the `PriceCache` writes, and the 500ms sleep loop, but contains
no GBM math itself:

```python
class SimulatorDataSource(MarketDataSource):
    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        for ticker in tickers:                              # seed the cache immediately
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def _run_loop(self) -> None:
        while True:
            try:
                if self._sim:
                    for ticker, price in self._sim.step().items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")   # one bad tick doesn't kill the loop
            await asyncio.sleep(self._interval)
```

Keeping the pure-math `GBMSimulator` separate from the asyncio/cache-wiring
`SimulatorDataSource` is what makes the 17 unit tests in
`test_simulator.py` possible without touching asyncio or a real `PriceCache` at
all — they construct a `GBMSimulator` directly and assert on `step()`'s output
(price bounds, correlation behavior, seeding, dynamic add/remove). The 10 tests
in `test_simulator_source.py` then cover the async wiring layer separately.

## File Structure (as built)

```
backend/
  app/
    market/
      simulator.py       # GBMSimulator + SimulatorDataSource
      seed_prices.py      # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, correlation constants
  tests/
    market/
      test_simulator.py         # GBMSimulator unit tests (17 tests, 98% coverage)
      test_simulator_source.py  # SimulatorDataSource integration tests (10 tests)
```

## Behavior Notes

- Prices can never go negative — GBM is multiplicative (`price *= exp(...)`), and
  `exp()` is always positive regardless of the random draw.
- The correlation matrix must be positive semi-definite for `np.linalg.cholesky`
  to succeed; every coefficient used here (`0.3`, `0.5`, `0.6`) is a fixed
  constant well inside valid correlation range, so this can't fail at runtime —
  it would only become a risk if correlations were ever made ticker-pair-specific
  and set inconsistently (e.g. `corr(A,B) = 0.9` and `corr(A,C) = corr(B,C) = -0.9`
  can produce a non-PSD matrix). The current scheme, which derives every pairwise
  value from a small set of shared group constants, avoids that entirely.
- `step()` is the hot path, called twice a second per running instance — it's
  kept allocation-light (no per-call list comprehensions beyond the final result
  dict) and uses NumPy's vectorized `standard_normal(n)` for the independent
  draws rather than drawing `n` individual `random.gauss()` calls.
- A live terminal demo of the simulator (Rich-based dashboard with sparklines,
  color-coded direction arrows, and an event log) is available at
  `backend/market_data_demo.py` — see `planning/MARKET_DATA_SUMMARY.md`.
