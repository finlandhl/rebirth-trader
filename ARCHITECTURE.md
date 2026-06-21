# Rebirth Trader — System Architecture Reference

**Version 3.0.0** | **Production Ready** | **June 21, 2026**

Complete architectural reference for the Rebirth Trader Binance Futures scalping system: initialization, data flow, position lifecycle, exit logic (6-layer + loss recovery trail + candle gate), state persistence, shutdown sequence, and error handling.

---

## Table of Contents

1. [System Overview & Data Flow](#1-system-overview--data-flow)
2. [Core Design Principles](#2-core-design-principles)
3. [Entry Flow & Initialization](#3-entry-flow--initialization)
4. [CLI Flags Reference](#4-cli-flags-reference)
5. [Stream Data Processing & Price/Candle Buffers](#5-stream-data-processing--pricecandle-buffers)
6. [Signal Generation & Market Assessment](#6-signal-generation--market-assessment)
7. [Position Lifecycle](#7-position-lifecycle)
8. [Position Sizing & Capital Management](#8-position-sizing--capital-management)
9. [Exit Logic System (6-Layer Priority + Loss Recovery Trail)](#9-exit-logic-system-6-layer-priority--loss-recovery-trail)
10. [Candle Gate — Profit & Loss Trail Suppression](#10-candle-gate--profit--loss-trail-suppression)
11. [Hedge Management](#11-hedge-management)
12. [Trading Pause Window](#12-trading-pause-window)
13. [Symbol Blacklist (`--blacked`)](#13-symbol-blacklist---blacked)
14. [State Persistence](#14-state-persistence)
15. [Signal Handler & Shutdown Sequence](#15-signal-handler--shutdown-sequence)
16. [Auto-Mode & Systemd Integration](#16-auto-mode--systemd-integration)
17. [Error Handling & Recovery](#17-error-handling--recovery)

---

## 1. System Overview & Data Flow

```
Binance Futures (Hedge Mode)
    ↕ WebSocket streams (up to 536 USDT/USDC pairs)
streamer.py
    ↓ CLI args parsed
RebirthTraderCLI (cli.py)
    ↓ HFScalper instance
HFSCalper (hfscalper.py)
    ↓ per-position coroutines         ↓ --ui flag
Position.monitor() × 20 max      UI Server (port 3000, 0.0.0.0)
    ↓ exit logic loop                ↓ nginx proxy (/dashboard/ → /)
Binance API (close)               ↓ cloudflared (--tunnel)
                              TryCloudflare URL
```

### Data Pipeline

```
WS kline/ticker/aggTrade
    → process_stream_data()                    # Parse + validate price
    → price_buffer[symbol] (deque maxlen=120)  # Zero-API price storage
    → candle_buffer[symbol] (deque maxlen=100) # Filtered 1m close prices only
    → (every 5 messages) assess_market_conditions()
        → EMA 50/100 trend detection
        → RSI 14 filter
        → Volatility check
        → Confidence scoring (0.0-1.0)
        → Candle trend confirmation (EMA 12/26 at candle_kline_interval)
    → calculate_position()                      # Size + capital validation
    → open_position()                            # Place order on Binance
    → Position.monitor() coroutine               # Independent exit loop
```

---

## 2. Core Design Principles

### Position-Centric Architecture
- Each position runs in its own `asyncio` coroutine with independent error isolation
- Failures in one position never cascade to others
- All positions share resources via callback pattern (`_get_price`, `_close_position`, etc.)
- Up to 20 concurrent positions (hedges excluded from limit)

### Price Buffer Pattern
- WebSocket price data cached in memory — zero Binance API calls for monitoring prices
- `price_buffer[symbol]` = `deque(maxlen=120)` populated by `process_stream_data()`
- Positions retrieve prices via `_get_price()` callback — buffer first, API fallback on staleness
- `BUFFER_STALENESS_THRESHOLD = 30s`, `STREAM_UNHEALTHY_THRESHOLD = 60s`

### Candle Buffer Pattern
|- Separate buffer for trend-confirmation only (not price retrieval)
|- `candle_buffer[symbol]` = `deque(maxlen=100)` populated by closed 5m candles (REST-seeded at startup)
|- Filtered by `candle_kline_interval` — only candles matching this interval are appended
|- Cold start eliminated: `_seed_candle_buffer()` fetches `candle_buffer_maxlen` historical 5m klines from Binance REST API per symbol before streaming begins (Semaphore 20 for concurrency, ~5s for 535 symbols)
|- Used by both exit candle gate and entry candle confirmation

### No `--demo` Flag
- Trading mode auto-detected: `is_live_trading = (client is not None)`
- No `--live` flag = console mode (simulated trading, no Binance orders)
- `--live` flag = real orders on Binance Futures

### Singleton Configuration
- All risk parameters defined once in `HFScalper.__init__()`
- Propagated to positions via callbacks and dataclass defaults
- Single change point for stop loss, target, leverage, etc.

---

## 3. Entry Flow & Initialization

### Call Chain
```
streamer.py:main()
    → parse CLI args (including --persist systemd setup)
    → CLIArgs dataclass wrapper
    → RebirthTraderCLI().run(cli_args)
        → setup_signal_handlers()              # SIGINT + SIGTERM
        → display startup banner
        → check --persist for state restoration
        → initialize_scalper()                  # Creates HFScalper
            → HFScalper.__init__()
        → _restore_from_state() if persist      # 2-pass restoration
        → _seed_candle_buffer()                  # Pre-fill candle buffers from REST (before streaming)
        → streamer.start()                      # Begin WebSocket streaming
        → main loop (until Ctrl+C / signal)
            → cli._run_persist()                # Periodic state saves
        → cleanup()                             # Shutdown sequence
```

### HFScalper.__init__() — Key Initializations

**File:** `hfscalper.py:1455`

1. **Trading mode detection:**
   ```python
   self.is_live_trading = client is not None
   ```

2. **Risk parameters:**
   - `fee_rate_total = 0.0008` (0.08%)
   - `min_notional_threshold = 5.0` ($5 min)
   - `risk_per_trade = 0.027` (2.7%)
   - `max_risk_adjusted = 0.028` (2.8% max risk after confidence adjustment)
   - `max_capital_per_position = 0.029` (2.9%)
   - `max_total_capital_usage = 0.90` (90%)
   - `hard_stop_loss = 0.04` (4%)
   - `profit_target = 0.06` (6%)
   - `trail_activation = 0.025` (2.5%)
   - `trail_distance = 0.00092` (0.092%)
   - `loss_trail_activation = 0.035` (3.5%)
   - `loss_trail_distance = 0.011` (1.1%)
   - `max_holding_minutes = 10080` (7 days)

3. **Leverage tiers:**
   - `max_leverage = 30` (confidence > 0.8)
   - `default_leverage = 20` (0.5 < conf ≤ 0.8)
   - `min_leverage = 10` (conf ≤ 0.5)

4. **Trading parameters:**
   - `max_concurrent_positions = 20` (hedges excluded)
   - `trend_threshold = 0.0005` (0.05%)
   - `volatility_threshold = 2.5`
   - `confidence_normalization = 0.0006`
   - `confidence_threshold = 0.20`
   - `signal_check_interval = 5`

5. **EMA / buffer parameters:**
   - `ema_slow_period = 100`
   - `buffer_threshold = 110`
   - `price_buffer_maxlen = 120`
   - `candle_buffer_maxlen = 100` (~500 min of 5m candle history)
   - `candle_ema_fast = 12`
   - `candle_ema_slow = 26`
   - `candle_trend_threshold = 0.001` (0.1%)
   - `candle_kline_interval = '5m'`

6. **Trading pause window:**
   - `trading_pause_start_hour = 23` (11pm UTC)
   - `trading_pause_end_hour = 2` (2am UTC)
   - Blocks new entries and hedges during pause window

7. **Symbol blacklist:**
   - `blacked_symbols: Dict[str, datetime]` — symbol → expiry
   - `blacked_perf: Dict[str, dict]` — consecutive loss tracking
   - `blacked_max_consecutive_losses = 3`
   - `blacked_cooldown_minutes = 1440` (24h, overridable via `--blacked NH`)

8. **Concurrency control:**
   ```python
   self.opening_positions: Set[str] = set()
   self.opening_locks: Dict[str, asyncio.Lock] = {}
   self.position_lock = asyncio.Lock()
   ```

9. **Stream health tracking:**
   ```python
   self.last_real_price_update: Dict[str, datetime] = {}
   ```

10. **Persistence startup:**
    ```python
    if persist:
        self._start_persist()
    ```

---

## 4. CLI Flags Reference

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--scalp` | store_true | False | Enable HFScalper trading mode |
| `--live` | store_true | False | Real money trading on Binance Futures |
| `--persist` | store_true | False | Save/restore state + install systemd service |
| `--auto` / `-a` | store_true | False | Bypass all confirmations (for systemd/automation) |
| `--loghfs` | store_true | False | Enable JSONL event logging |
| `--ui` | store_true | False | Start embedded web dashboard on port 3000 (0.0.0.0) |
| `--tunnel` | store_true | False | Create Cloudflare Tunnel for public internet access (requires `--ui`) |
| `--symbols` | str | None | Comma-separated symbol list |
| `--all` | store_true | False | All tradable futures symbols |
| `--raw` | store_true | False | Save raw WebSocket data |
| `--save` | str | None | Save stream to file |
| `--blacked` | str[*] | None | Symbol blacklist: 3 consecutive losses → N-hour auto-suspend (default 24H). Pre-blacklist symbols (comma-sep) and/or override cooldown (e.g. 5H). Persisted across restarts. |
| `--version` | - | - | Show version number and exit |
| `-h, --help` | - | - | Show this help message and exit |

**Mutual exclusion:** `--symbols` and `--all` are mutually exclusive. Exactly one is required.

---

## 5. Stream Data Processing & Price/Candle Buffers

### process_stream_data() — `hfscalper.py:3618`

Called by `streamer.py` for every kline, ticker, and trade message.

**Flow:**
1. Extract price from ticker (`c`), trade (`p`), or kline close (`k.c`)
2. Reject invalid prices (`price <= 0`)
3. Update `last_real_price_update[symbol]` timestamp
4. Populate `price_buffer[symbol]` (deque, maxlen=120)
5. For kline data: populate `candle_buffer[symbol]` only when the kline's `i` (interval) matches `candle_kline_interval` AND `x` (candle closed) is True
6. Update position `current_price` directly (zero-latency feed)
7. Calculate rolling volatility from price returns
8. Every `signal_check_interval` (5) messages: call `assess_market_conditions()`

**Candle buffer filter (line 3639–3640):**
```python
# Populate candle buffer on close at candle_kline_interval (for reversal confirmation)
if kline_data.get('x') and kline_data.get('i') == self.candle_kline_interval:
    self.candle_buffer[symbol].append(current_price)
```

Only 5m closed candles populate the buffer (no mixing from other intervals).

**Cold start elimination (June 19):** Before WebSocket streaming begins, `_seed_candle_buffer()` fetches 100 historical 5m klines per symbol via Binance REST API (`GET /fapi/v1/klines`) using `asyncio.Semaphore(20)` for controlled concurrency. All candle buffers are full before the first WebSocket candle close arrives — EMA(12)/EMA(26) is immediately computable. Seed events logged as `candle_buffer_seed` (per-symbol) and `candle_buffer_seed_failed` on errors.

### Price Validation
- Null/zero prices rejected — prevents NaN propagation
- `last_real_price_update` tracked separately from WebSocket heartbeat messages
- Prevents opening positions on dead streams

### Price Retrieval: `_get_price()`

```python
def _get_price(symbol):
    1. Check price_buffer[symbol] for data < 30s old → return it
    2. If stale (<60s) OR live mode: API fallback via Binance
    3. If no data at all: return None
```

Constants:
- `MAX_API_TIMEOUT = 20.0` seconds
- `BUFFER_STALENESS_THRESHOLD = 30.0` seconds
- `STREAM_UNHEALTHY_THRESHOLD = 60.0` seconds

---

## 6. Signal Generation & Market Assessment

### assess_market_conditions()

Called every 5 WebSocket messages per symbol (after buffer threshold of 110 entries).

**Prerequisites:**
- Price buffer must have ≥ 110 entries (ema_slow + 10 margin)

**Assessment steps:**

1. **Trend detection** — EMA 50/100 crossover:
   ```python
   ema_fast = EMA(50, prices)
   ema_slow = EMA(100, prices)
   trend = "bullish" if ema_fast > ema_slow else "bearish"
   trend_strength = abs(ema_fast / ema_slow - 1)
   trend_valid = trend_strength >= 0.0005  # 0.05% trend_threshold
   ```

2. **RSI 14 filter:**
   - RSI > 75: overbought (bearish signal)
   - RSI < 25: oversold (bullish signal)
   - Extreme RSI overrides EMA trend

3. **Volatility filter:**
   - `volatility_threshold = 2.5` — skip if exceeded

4. **Confidence scoring:**
   ```python
   confidence = min(trend_strength / 0.0006, 1.0)
   ```

5. **Leverage selection:**
   - confidence > 0.7 AND volatility < 1.0 → 30x
   - confidence > 0.5 AND volatility < 1.5 → 20x
   - else → 10x

6. **Direction and entry signal:**
   ```python
   direction = "long" if trend == "bullish" and rsi_filter != "overbought" else "bearish"
   should_enter = trend_valid and rsi_filter != "overextreme"
   ```

7. **Kline confirmation** (line 2089–2104): Verifies 1m candle EMA(12/26) at `candle_kline_interval` agrees with the tick trend before allowing entry. If candle trend disagrees → entry blocked. If insufficient candle data (< 26 entries in candle_buffer) → entry allowed anyway (cold start grace).

---

## 7. Position Lifecycle

### Lifecycle States

```
IDLE → PLACEHOLDER → OPENING (order pending) → ACTIVE (monitoring) → CLOSED
```

### Detailed Flow: `open_position()`

#### Step 1: Pre-checks
- **Stream health:** `last_real_price_update` must be < 60s old
- **Duplicate gate:** Check `opening_positions` set
- **Position limit:** Count active positions, skip if ≥ max_concurrent_positions (hedges exempt)
- **Capital check:** `available_capital >= capital_required`

#### Step 2: Placeholder Reservation
```python
placeholder = PositionPlaceholder()
self.active_positions.setdefault(symbol, []).append(placeholder)
```
Placeholder is **falsy** (`__bool__` returns `False`) — correctly handled by state persistence filters and position-count logic.

#### Step 3: Order Placement
- Set leverage on Binance
- Set margin type — sends `"CROSSED"` (known bug: should be `"CROSS"`)
- Place MARKET order via Binance API
- Handle `-4164` auto-scale: if order fails with notional < $5, rescale quantity and retry once
- Wait for fill confirmation — up to 30s

#### Step 4: Position Creation
```python
position = Position(
    symbol=symbol,
    direction=direction,        # "long" or "short"
    entry_price=avg_fill_price,
    stop_price=entry * (1 - 0.04 if long else 1 + 0.04),  # 4% hard stop
    target_price=entry * (1 + 0.06 if long else 1 - 0.06), # 6% target
    coin_position=coin_qty,
    capital_required=capital_used,
    position_id=self._next_position_id(),
    is_hedge=is_hedge,
    # ... plus all exit params, timestamps
)
```

#### Step 5: Placeholder Replacement
Under `position_lock`: replace placeholder at the same list index with the real Position object.

#### Step 6: Callback Attachment
```python
position._get_price = self._get_price
position._close_position = self._close_position
position._detect_reversal = self._detect_reversal
position._check_hedge = self._check_hedge
position.logger = self.logger
position.hfscalper = self
```

#### Step 7: Monitor Coroutine Start
```python
monitor_task = asyncio.create_task(position.monitor())
```

### Monitor Loop: `Position.monitor()`

Runs as an independent coroutine until `position.is_closed = True`.

```python
while not self.is_closed:
    # 1. Get price (buffer → API fallback → stale fallback)
    current_price = await self._get_price(self.symbol)

    # 2. Update unrealized PnL
    if current_price:
        self.current_price = current_price
        self.unrealized_pnl = (current_price / self.entry_price - 1)
        if self.direction == "short":
            self.unrealized_pnl = -self.unrealized_pnl

    # 3. Reversal detection (every cycle)
    reversal_signal = await self._detect_reversal(self.symbol)

    # 4. Candle gate (S2) — check trend before trail activation
    self._candle_skip_trail = False  # Reset each cycle
    profit_check = self.unrealized_pnl >= self.trail_activation * self.candle_trail_min_ratio
    loss_check = self.unrealized_pnl < 0 and abs(self.unrealized_pnl) >= self.loss_trail_activation

    if not is_hedge and (profit_check or loss_check) and hfscalper is not None:
        candle_trend = hfscalper._assess_candle_trend(self.symbol)
        if candle_trend.get('valid') and candle_trend['direction'] == self.direction:
            self._candle_skip_trail = True  # Same trend → suppress trail

    # 5. Check exit logics (priority order)
    exit_signal = self.check_exit_logics(current_price, reversal_data)

    # 6. If exit triggered → close_position()
    if exit_signal.should_exit:
        success = await self.close_position(current_price, exit_signal)

    # 7. Wait 1 second
    await asyncio.sleep(1)
```

**Error isolation:** Every step wrapped in try/except — errors are logged, position continues.

---

## 8. Position Sizing & Capital Management

### Sizing Flow: `calculate_position()`

```
adjusted_risk = capital * risk_per_trade  # 2.7% of available
leverage from confidence tier            # 10x, 20x, or 30x
position_value = adjusted_risk * leverage

# Min notional check
if position_value < min_notional * 1.1:   # $5 × 1.1 = $5.50
    position_value = min_notional * 1.1

# Coin quantity
coin_position = position_value / current_price
if coin_position < symbol_min_qty:
    coin_position = symbol_min_qty

# Capital required
capital_required = position_value / leverage

# Per-position cap (2.9% of capital)
if capital_required > capital * 0.029:
    capital_required = capital * 0.029
```

### Capital Gates

| Check | Threshold | Effect |
|-------|-----------|--------|
| Per-position cap | 2.9% | Caps single position capital |
| Total usage cap | 90% | Caps sum of all active positions |
| Min notional | $5 × 1.1 | Binance minimum + 10% buffer |
| Available capital | remaining | Deducted on open, returned on close |
| Balance cap (live) | $1000 | Live accounts capped even with larger balance |

---

## 9. Exit Logic System (6-Layer Priority + Loss Recovery Trail)

### ExitSignal Dataclass

```python
@dataclass
class ExitSignal:
    should_exit: bool = False
    exit_price: float = 0.0
    exit_reason: str = ""
    logic_name: str = ""  # 'stop_loss', 'take_profit', 'trailing_stop', 'time_limit', 'reversal'
```

### Priority Order

Exit logics are registered and checked in strict priority order:

| Priority | Logic | Class | Trigger |
|----------|-------|-------|---------|
| 1 | **Stop Loss + Loss Recovery Trail** | `StopLossExit` (line 108) | Two-stage: loss trail checks first, hard stop second |
| 2 | **Take Profit** | `TakeProfitExit` (line 186) | PnL ≥ +6% (price crosses `target_price`) |
| 3 | **Trailing Stop** | `TrailingStopExit` (line 253) | +1.3% peak then -0.092% pullback |
| 4 | **Time Limit** | `TimeLimitExit` | ≥ `max_holding_minutes` (10080 min = 7 days) |
| 5 | **Reversal** | `ReversalDetectionExit` (line 301) | Trend reversal signal with confidence |

### StopLossExit — Two-Stage Check (lines 108–184)

#### Stage 1: Loss Recovery Trail (checked FIRST, lines 130–159)

The loss-side mirror of the profit trailing stop:

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `loss_trail_activation` | 3.5% | Minimum drawdown before trail arms |
| `loss_trail_distance` | 1.1% | Recovery from trough needed to exit |

**Logic:**
- Tracks `trough_pnl` — the deepest loss % reached
- Activates when `abs(pnl) >= 3.5%`
- Once activated, measures **recovery from trough**: `trough_pnl - current_loss`
- If recovery >= 1.1%, triggers exit via `Loss Trail` signal
- **Gate support:** If `_candle_skip_trail` is set, the recovery check is suppressed (same candle trend → let loser recover)
- **Candle gate check order:** Before recovery check, the candle gate is consulted (lines 148–150)

#### Stage 2: Hard Stop (checked SECOND, lines 161–184)
Traditional fixed stop at `stop_price` (4% from entry). Always fires regardless of candle gate.

### Profit Trailing Stop (`TrailingStopExit`, lines 253-300)

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `trail_activation` | 2.5% | Profit threshold before trailing arms |
| `trail_distance` | 0.092% | Pullback from peak that triggers exit |

**Logic:**
- Tracks `peak_pnl` — maximum profit % reached
- Activates `trail_activated` when `pnl >= 1.3%`
- Once activated, measures pullback from peak: `peak_pnl - current_pnl`
- If pullback >= 0.092%, triggers exit
- **Gate support (line 270):** If `_candle_skip_trail` is True AND trail is not yet activated, the activation is suppressed — letting winners run

---

## 10. Candle Gate — Profit & Loss Trail Suppression

### Overview

The candle gate is a runtime check in `Position.update()` that compares the **1m candle trend** (EMA 12/26 at `candle_kline_interval`) against the **position direction**. It sets `_candle_skip_trail = True` to suppress trail activation when the candle trend still favors the position.

### Gate Activation Thresholds

| Side | Check | Runtime value |
|------|-------|---------------|
| **Profit** (TrailingStopExit) | `PnL ≥ trail_activation × candle_trail_min_ratio` | 2.5% × 0.3 = **0.75%** |
| **Loss** (StopLossExit loss trail) | `PnL < 0 AND abs(PnL) ≥ loss_trail_activation` | **3.5% loss** |

Below these thresholds → gate is skipped entirely.

### Gate Decision Matrix

| 1m Candle Trend vs Position | `_candle_skip_trail` | Effect on Profit Trail | Effect on Loss Trail |
|-----------------------------|----------------------|------------------------|---------------------|
| **Same direction** | **True** → suppressed | TrailingStopExit cannot activate — winner runs higher | LossRecoveryTrail cannot trigger on recovery — loser bounces |
| **Opposite direction** | False → allowed | Trail activates normally — lock profit if pullback ≥ 0.092% | Loss trail checks recovery from trough — salvage if bounce ≥ 1.1% |
| Trend too weak / insufficient data | False → allowed | Same as opposite | Same as opposite |

### Scope limits
- **Profit trail:** Gate only blocks **activation** of the trailing stop (line 270: `if not self.trail_activated`). Once armed, the trail operates normally regardless of gate.
- **Loss trail:** Gate blocks the **recovery check** (lines 148-150 in StopLossExit). The **hard stop at 4% always fires** regardless.
- **Hedges:** Gate is skipped entirely for hedge positions (line 802).
- **Runtime only:** `_candle_skip_trail` is reset to `False` every monitor cycle and is **not persisted** to state.jsonl.

---

## 11. Hedge Management

### Hedge Mode Setup
- Binance Futures account set to **Hedge Mode** (`dualSidePosition=true`) at `binance_client.py:25`
- Supports both LONG and SHORT positions simultaneously per symbol
- Each position tracks `positionSide: "LONG"` or `"SHORT"`

### Defensive Hedge Gate
When a position experiences consecutive losses exceeding `defensive_hedge_minutes` (1440.0 = 24h):
1. Check if a candle trend exists for the symbol
2. If candle trend opposes the current position → open a defensive hedge
3. The hedge is the opposite direction of the current position

The hedge itself bypasses the `max_concurrent_positions` limit.

### Hedge Pair Linking
On state restoration (`_restore_from_state()`), positions are linked to their hedge counterpart via `_hedge_pair_id`. This enables combined P&L tracking and coordinated exit decisions.

---

## 12. Trading Pause Window

### Configuration
```python
self.trading_pause_start_hour = 23   # 11pm UTC
self.trading_pause_end_hour = 2      # 2am UTC
```

### Behavior
During the pause window (23:00–02:00 UTC):
- **New entries blocked:** `assess_market_conditions()` → `calculate_position()` chain returns early with `'reason': 'trading_pause'`
- **New hedges blocked:** `_check_hedge()` skips defensive hedge creation during pause window
- **Existing positions unaffected:** All active monitoring coroutines continue normally with exit logic, stop loss, trailing stop, time limit — only new position creation is blocked
- **Logged:** Events logged as `'entry_skipped_pause'` and `'hedge_skipped_pause'`

### Enforcement Points
- `process_stream_data()` at signal evaluation: checks `now_hour >= start or now_hour < end`
- `_check_hedge()`: same check before opening defensive hedges

---

## 13. Symbol Blacklist (`--blacked`)

### What It Does
Auto-suspends trading on symbols that show repeated consecutive losses, preventing the bot from burning capital on consistently failing pairs.

### Configuration (from HFScalper.__init__)
```python
self.blacked_symbols: Dict[str, datetime] = {}           # symbol → cooldown expiry
self.blacked_perf: Dict[str, dict] = {}                  # symbol → {cons_losses, total_pnl}
self.blacked_max_consecutive_losses = 3                   # trigger threshold
self.blacked_cooldown_minutes = 1440                      # 24h default (overridable)
```

### CLI: `--blacked [SYM1,SYM2 [NH]]`
| Part | Description |
|------|-------------|
| (no args) | Enable blacklist with default 24h cooldown |
| `SYM1,SYM2` | Pre-blacklist symbols on startup (comma-separated) |
| `NH` | Override cooldown duration (e.g. `5H` = 5 hours, `48H` = 48 hours) |

### Loss Tracking
```python
# After each position close with loss:
perf = self.blacked_perf.setdefault(symbol, {'cons_losses': 0, 'total_pnl': 0.0})
perf['cons_losses'] += 1
perf['total_pnl'] += pnl

# If threshold reached:
if perf['cons_losses'] >= self.blacked_max_consecutive_losses:
    expiry = datetime.now() + timedelta(minutes=self.blacked_cooldown_minutes)
    self.blacked_symbols[symbol] = expiry
```

### Enforcement Points
- **New entries** (process_stream_data): if `symbol in blacked_symbols`, entry skipped with `'entry_skipped_blacked'` event
- **Defensive hedges** (_check_hedge): if `symbol in blacked_symbols`, hedge skipped with `'hedge_skipped_blacked'` event

### Persistence

Blacklist state is saved and restored via `state.jsonl`:
- **Save:** `blacked_symbols` dict serialized as `{sym: iso_expiry}`
- **Restore:** `_restore_from_state()` deserializes back to `datetime` objects

### Cooldown Expiry

On each `assess_market_conditions()` call, expired entries are cleaned up:
```python
if symbol in self.blacked_symbols and datetime.now() >= expiry:
    del self.blacked_symbols[symbol]
    self.blacked_perf.pop(symbol, None)
```

---

## 14. Dashboard & Monitoring

### Architecture

```
streamer.py (--ui flag)
    ↓
UI Server (server.py) on 0.0.0.0:3000
    ├── GET /          → ui/index.html (dashboard)
    ├── GET /api/stats      → live aggregated stats
    ├── GET /api/equity     → equity curve data
    ├── GET /api/positions  → active position details
    ├── GET /api/history    → closed trade history
    └── GET /api/journal    → raw full journal
    ↓ nginx proxy (port 80)
├── /dashboard/ → proxied to http://127.0.0.1:3000/
└── /api/       → proxied to http://127.0.0.1:3000/api/
    ↓ cloudflared tunnel (--tunnel flag)
https://<random>.trycloudflare.com → localhost:3000
```

### UI Server (`src/rebirth_trader/ui/server.py`)

- **Fast:** Pure Python HTTP server with zero dependencies (stdlib http.server)
- **Bind:** `0.0.0.0` (LAN accessible) when `--ui` is active
- **State source:** Reads last line of `logs/state.jsonl` on each request
- **Trade journal:** Reads `logs/history.jsonl` independently (decoupled from `--loghfs`)
- **Dashboard template:** `src/rebirth_trader/ui/index.html` read at startup — requires bot restart for HTML changes

### Dashboard Template (`ui/index.html`)

Single-file HTML with embedded CSS and JavaScript. Features:
- **Stat cards:** Glass-morphism cards with colored accent bars
- **Equity curve:** Chart.js canvas with white gradient fill
- **Positions table:** Live 22+ positions with uPnL coloring
- **Best/Worst tables:** Top performers by P&L
- **Streak dots:** Visual trade streak history (last 100 trades)
- **Trade journal:** Last 1000 closed trades with search
- **Auto-refresh:** 10-second interval (matches state write cycle)

### Cloudflare Tunnel (`--tunnel`)

When `--tunnel` is passed with `--ui`:

1. **Detection:** Checks `/usr/local/bin/cloudflared`
2. **Auto-install:** Downloads cloudflared from Cloudflare if missing
3. **Spawn:** Runs `cloudflared tunnel --url http://localhost:3000` as subprocess
4. **URL parsing:** Reads the TryCloudflare URL from stdout and prints it
5. **Cleanup:** Kills the tunnel subprocess on bot shutdown

**Requirements:** Outbound internet access only — no inbound ports needed beyond existing UFW rules.

### Nginx Proxy

The production nginx config at `/etc/nginx/sites-available/rebirth-trader` proxies:
- `/dashboard/` → `http://127.0.0.1:3000/` (dashboard HTML)
- `/api/` → `http://127.0.0.1:3000` (API endpoints)

The landing page (`/var/www/html/index.html`) links to `/dashboard/` and fetches `/api/stats` for live stats.

### Dashboard Data Model

| Stat | Source Field | Description |
|------|-------------|-------------|
| Wallet Balance | `wallet_balance` | Current wallet from state.jsonl |
| Available Capital | `available_capital` | Unused capital |
| Init Balance | `initial_balance` | Starting capital |
| Total P&L | `total_pnl` | Realized + unrealized |
| Unrealized P&L | `unrealized_pnl` | Open position floating P&L |
| Total Positions | `total_positions_created` | Cumulative entry count |
| Win Rate | computed | wins / (wins + losses) |
| Total Fees | `total_fees` | Cumulative fee spend |

---

## 15. State Persistence

### Architecture

```
Periodic (every 10s):
    Position.monitor() → _periodic_save() → save_state() → logs/state.jsonl

On Shutdown:
    cleanup() Step 2 → HFScalper.save_state() → final snapshot

On Restart:
    _has_state() → _load_last_state() → _restore_from_state() → monitor
```

### save_state()

```python
snapshot = {
    "version": 1,
    "available_capital": self.available_capital,
    "total_positions_created": self.total_positions_created,
    "active_positions": filtered_positions(skip_placeholders),
    "closed_positions": last_500_closed,
    "timestamp": datetime.now().isoformat()
}
```

- **JSONL append-only** — crash-safe: previous line preserved if write fails
- **Placeholder filter:** `PositionPlaceholder` objects excluded (their `__bool__` returns False)
- **File I/O:** Uses `asyncio.to_thread()` for non-blocking writes

### _has_state()
Static method checks if `logs/state.jsonl` exists and is non-empty.

### _load_last_state()
Static method — seeks to last line via `f.seek(-2, os.SEEK_END)`, avoiding full file load.

### _restore_from_state()

Two-pass restoration:
1. **Pass 1:** Create `Position` objects for each active position, attach callbacks
2. **Pass 2:** Link hedge pairs by `_hedge_pair_id`

Then start `asyncio.create_task(position.monitor())` for each non-closed position.

**Key restoration behaviors:**
- `max_holding_minutes` overridden with current HFScalper value (not stale saved value)
- All exit parameters use `data.get(..., Position.<default>)` for backward compatibility
- `_candle_skip_trail` is a runtime-only field (not persisted — always starts as False)

---

## 16. Signal Handler & Shutdown Sequence

### Signal Handler Setup — `cli.py:83`

```python
def setup_signal_handlers(self):
    loop = asyncio.get_event_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(
            sig,
            lambda s=sig: self._handle_signal(s)
        )
```

Two handler paths based on mode:

**Auto Mode** (`--auto` or `REBIRTH_TRADER_SYSTEMD=1`):
```python
def _handle_signal(self, sig):
    self.running = False  # Immediate clean shutdown
    # Bypasses pause menu entirely
```

**Interactive Mode:**
```python
def _handle_signal(self, sig):
    self.paused = True     # Triggers pause menu on next loop iteration
```

### cleanup() Shutdown Sequence

| Step | Action | Timeout | Description |
|------|--------|---------|-------------|
| 0a | State save | - | Final snapshot written before connections close |
| 1 | Position close | 300s | `hfscalper.close()` — close all open positions |
| 2 | State save | - | Final state snapshot after position closure |
| 3 | Final summary | - | Print summary stats |
| 4 | WebSocket close | 120s | `streamer.stop_all()` |
| 5 | HTTP client close | 120s | `client.close()` |
| 6 | File close | 60s | Raw/output files |
| 7 | Bytecode cleanup | 60s | `__pycache__` removal |

**Auto-mode skip:** Positions are NOT closed on shutdown in auto mode — they remain open on Binance, and the bot restores them on next start via `--persist`.

---

## 17. Auto-Mode & Systemd Integration

### Auto-Mode Detection

The bot enters auto-mode when **either** condition is true:
- `--auto` flag passed on CLI
- `REBIRTH_TRADER_SYSTEMD=1` environment variable set

**Effects of auto-mode:**
- Bypasses live trading confirmation prompt
- Signal handler sets `running = False` directly (no pause menu)
- Pause menu auto-selects 'y' (close all + shutdown)
- Position close skipped in cleanup() — preserves positions for next session

### Systemd Service

Auto-installed on first `--persist` run. Idempotent — checks `systemctl is-enabled` before installing.

**Unit file** (`/etc/systemd/system/rebirth-trader.service`):
```ini
[Unit]
Description=Rebirth Trader Crypto Trading Bot
After=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/kali/rebirth-trader
ExecStart=/root/.local/bin/uv run streamer.py --scalp --all --live --persist --auto --ui --tunnel
Restart=always
RestartSec=10
Environment=REBIRTH_TRADER_SYSTEMD=1
StandardOutput=null
StandardError=null

[Install]
WantedBy=multi-user.target
```

**Key behaviors:**
- `Restart=always` — restarts 10s after any crash
- `REBIRTH_TRADER_SYSTEMD=1` — prevents recursive systemd setup, auto-confirms
- No TTY — process survives SSH disconnect
- `WantedBy=multi-user.target` — starts on boot
- Recursion guard — `_setup_systemd_service()` exits early if `REBIRTH_TRADER_SYSTEMD` is set

---

## 18. Error Handling & Recovery

### Error Classification

| Severity | Examples | Response |
|----------|----------|----------|
| 🔴 Critical | State file corruption, WS disconnect | Logged, position monitoring continues via API fallback |
| 🟠 Major | Order failure (-4003, -4164, -2013) | Capital returned, placeholder cleaned, logged |
| 🟡 Minor | Logger exception, stale buffer | Try/except, execution continues |
| 🔵 Info | Leverage set failure, duplicate skip | Logged, no action needed |

### Error Recovery Patterns

**1. Order failure → capital return:** If `open_position()` fails at any point after capital deduction, capital is returned to `available_capital` and placeholder is removed from `active_positions`.

**2. Price buffer staleness:** `_get_price()` uses 3-tier fallback: buffer (fresh) → API (live mode) → stale buffer (last resort). Positions continue monitoring with the best available data.

**3. Logger exception isolation:** All `logger.log()` calls wrapped in try/except — a logging failure never crashes a position.

**4. Stream health validation:** `last_real_price_update` tracks actual price data (not WS heartbeat). If > 60s stale, `open_position()` refuses to open new positions.

**5. Emergency close timeout:** If exit triggered but position not closed after 30s, `monitor()` force-closes via direct Binance API call.

**6. systemctl stop hang:** Pre-existing asyncio signal propagation issue under systemd. Use `systemctl kill -s SIGKILL` to force-stop.
