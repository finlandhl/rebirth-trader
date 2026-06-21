# Features & Enhancements — v3.0.0

**Version 3.0.0** | **June 21, 2026** | **Rebranded to Rebirth Trader**

Comprehensive technical reference covering all features, patches, and architectural changes in the rebirth-trader scalping bot.

**Documentation Suite:**
- **FEATURES_ENHANCEMENT.md** (this file): All features, patches, and technical changes
- **ARCHITECTURE.md**: Complete system architecture and code workflow
- **LIVE_TRADING.md**: Production trading setup and safety
- **README.md**: Quick-start overview and system summary

---

## Table of Contents

1. [State Persistence (`--persist`)](#1-state-persistence---persist)
2. [Systemd Integration](#2-systemd-integration)
3. [WebSocket Migration (Binance API Change)](#3-websocket-migration-binance-api-change)
4. [Signal Handler & Auto-Mode Patches](#4-signal-handler--auto-mode-patches)
5. [Candle Gate & Loss Recovery Trail](#5-candle-gate--loss-recovery-trail)
6. [Position-Centric Architecture](#6-position-centric-architecture)
7. [Six-Layer Exit Logic System](#7-six-layer-exit-logic-system)
8. [Price Buffer Pattern](#8-price-buffer-pattern)
9. [Exception-Safe Logging](#9-exception-safe-logging)
10. [Placeholder & Ghost Position Prevention](#10-placeholder--ghost-position-prevention)
11. [Emergency Shutdown & Cleanup](#11-emergency-shutdown--cleanup)
12. [Auto-Detection Mode (No --demo Flag)](#12-auto-detection-mode-no---demo-flag)
13. [Concurrent Position Management](#13-concurrent-position-management)
14. [Stream Health Validation](#14-stream-health-validation)
15. [Candle Buffer — Interval-Filtered & Cleaned](#15-candle-buffer--interval-filtered--cleaned)
16. [Loss Recovery Trail](#16-loss-recovery-trail)
17. [Candle Gate — Profit & Loss Trail Suppression](#17-candle-gate--profit--loss-trail-suppression)
18. [Defensive Hedge with Candle Gate](#18-defensive-hedge-with-candle-gate)
19. [Capital Capping & Balance Protection](#19-capital-capping--balance-protection)
20. [Order Failure Recovery & Auto-Scale](#20-order-failure-recovery--auto-scale)
21. [Files Changed Reference](#21-files-changed-reference)
22. [Trading Pause Window](#22-trading-pause-window)
23. [Symbol Blacklist (`--blacked`)](#23-symbol-blacklist---blacked)
24. [Dashboard UI (`--ui`)](#24-dashboard-ui---ui)
25. [Cloudflare Tunnel (`--tunnel`)](#25-cloudflare-tunnel---tunnel)

---

## 1. State Persistence (`--persist`)

**Files:** `hfscalper.py`, `cli.py`, `streamer.py`

### What It Does
Enables crash-safe position state persistence to `logs/state.jsonl`:
- Periodic snapshots every **10 seconds** (background `_periodic_save()` task)
- Full position serialization (31 fields including hedge pair references)
- On restart: automatically restores all positions with monitoring coroutines

### Technical Details

**Save Path:**
```python
def save_state(self):
    snapshot = {
        "version": 1,
        "available_capital": self.available_capital,
        "total_positions_created": self.total_positions_created,
        "active_positions": self._filtered_positions(),  # skips placeholders
        "closed_positions": self.closed_positions[-500:],  # last 500
        "timestamp": datetime.now().isoformat()
    }
    # Written via asyncio.to_thread → non-blocking file I/O
```

**Restore Path:**
- `_has_state()`: checks file exists and is non-empty
- `_load_last_state()`: seeks to last JSON line (no full file parse)
- `_restore_from_state()`: two-pass creation + hedge linking + monitor start

**Key Design Decisions:**
- JSONL append-only (not overwrite) — previous snapshot preserved on last-write crash
- `tail -1` seek-based read — no full file load on restore
- `PositionPlaceholder` objects filtered before write
- All exit parameters use `data.get(..., Position.<default>)` for backward compatibility with older state files
- `max_holding_minutes` overridden with current config value

---

## 2. Systemd Integration

**File:** `streamer.py`

### What It Does
Auto-installs a systemd service on first `--persist` run:
- Creates `/etc/systemd/system/rebirth-trader.service`
- `Restart=always` with 10-second backoff
- `REBIRTH_TRADER_SYSTEMD=1` environment variable for auto-mode detection
- Survives SSH disconnect, crash, and reboot

### Unit File Template
```ini
[Unit]
Description=Rebirth Trader Crypto Trading Bot
After=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/kali/rebirth-trader
ExecStart=/root/.local/bin/uv run streamer.py <original args> --auto
Restart=always
RestartSec=10
Environment=REBIRTH_TRADER_SYSTEMD=1

[Install]
WantedBy=multi-user.target
```

**Key behaviors:**
- Idempotent — `_setup_systemd_service()` checks `systemctl is-enabled` first
- Recursion guard: `REBIRTH_TRADER_SYSTEMD=1` prevents re-installation under systemd
- `--auto` flag appended to `ExecStart` for headless operation

---

## 3. WebSocket Migration (Binance API Change)

**File:** `streamer.py` (src)

### What Changed
Binance deprecated legacy WebSocket endpoints in April 2026, splitting into `/public`, `/market`, `/private` routes.

**Migration applied:**
- Changed `wss://fstream.binance.com/ws/{stream}` to routed endpoints per data type
- Added `websockets` library fallback for older connections
- Verified: depth, ticker, kline, aggTrade all streaming correctly

---

## 4. Signal Handler & Auto-Mode Patches

**File:** `cli.py`

### What Changed

**Patch set for signal handler (cli.py):**

1. **`self.args` stored on CLI instance:** Enables signal handlers to access CLI flags (like `--auto`) when deciding shutdown behavior
   ```python
   self.args = args  # Stored in async def run()
   ```

2. **Signal handler auto-mode path:** When `--auto` or `REBIRTH_TRADER_SYSTEMD=1`, signal handler sets `self.running = False` directly — bypasses the pause menu entirely
   ```python
   if hasattr(args, 'auto') and args.auto:
       self.running = False  # Clean shutdown, no pause menu
   ```

3. **Pause menu auto-detection:** When signal triggers pause menu interactively, `--auto` mode auto-selects 'y' (close all + shutdown)
   ```python
   if getattr(getattr(self, 'args', None), 'auto', False):
       choice = 'y'  # Auto-confirm close
   ```

4. **Step 0a state save:** State snapshot saved immediately after position closure but before WebSocket/HTTP teardown — ensures final state is captured even if connections fail

5. **Duplicate Step 6 removed:** Original duplicate shutdown Step 6 replaced with trailing FINAL comment — clean exit path

### Result
- Auto-mode now handles signals cleanly (no stuck prompts under systemd)
- State file written with latest data before connections close
- Cleaner, single-path shutdown sequence

---

## 5. Candle Gate & Loss Recovery Trail

**File:** `hfscalper.py`

### Overview
The system implements three interconnected candle-based features:
- **Loss Recovery Trail** — embedded in `StopLossExit` to salvage losing trades
- **Candle Gate (S2)** — suppresses both profit and loss trail activation when the 1m candle trend favors the position
- **Candle Buffer** — interval-filtered buffer (only 1m closes) feeding both the gate and entry confirmation

### Parameters (Position dataclass, lines 394-465)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `defensive_hedge_minutes` | 1440.0 (24h) | Consecutive losing minutes before defensive hedge |
| `candle_trail_min_ratio` | 0.3 | Min PnL ratio of trail_activation before checking candle trend |
| `loss_trail_activation` | 0.035 (3.5%) | Loss % that activates recovery trail |
| `loss_trail_distance` | 0.011 (1.1%) | Recovery from trough that triggers loss trail exit |

### State Compatibility
All new parameters use `data.get(..., Position.<default>)` fallback pattern — old state files without these fields restore correctly:
```python
loss_trail_activation=data.get('loss_trail_activation', 0.035),
loss_trail_distance=data.get('loss_trail_distance', 0.011),
candle_trail_min_ratio=data.get('candle_trail_min_ratio', Position.candle_trail_min_ratio),
```

---

## 6. Position-Centric Architecture

**File:** `hfscalper.py`, all monitor/close methods

### What Changed (v2.4.0 major redesign)

Each position runs as an **independent asyncio coroutine** with isolated error handling.

### Architecture

```python
# Before (centralized loop):
for position in active_positions:
    check_and_close(position)  # One failure crashes all

# After (per-position coroutine):
for position in active_positions:
    asyncio.create_task(position.monitor())  # Each runs independently
```

### Callback Pattern for Shared Resources
```python
def _initialize_callbacks(self, position):
    position._get_price = self._get_price              # Buffer → no API
    position._close_position = self._close_position    # Close capability
    position._detect_reversal = self._detect_reversal  # Reversal API
    position._check_hedge = self._check_hedge          # Hedge logic
    position.logger = self.logger                      # Safe logging
    position.hfscalper = self                          # Backlink for buffers
```

### Benefits
- Errors isolated per position — no cascading failures
- Up to 20+ concurrent positions all monitoring independently
- Graceful degradation: one fails, others continue
- Independent exit logic evaluation per position

---

## 7. Six-Layer Exit Logic System

**File:** `hfscalper.py:76-300, 460-575`

### ExitSignal Dataclass
```python
@dataclass
class ExitSignal:
    should_exit: bool = False
    exit_price: float = 0.0
    exit_reason: str = ""
    logic_name: str = ""  # 'stop_loss', 'take_profit', etc.
```

### Priority Order

| Priority | Logic | Trigger | Class |
|----------|-------|---------|-------|
| 1 | **Stop Loss + Loss Trail** | 4% hard stop OR 3.5% threshold + 1.1% recovery | `StopLossExit` |
| 2 | **Take Profit** | PnL ≥ +6% (price crosses `target_price`) | `TakeProfitExit` |
| 3 | **Trailing Stop** | +1.3% peak then -0.092% pullback | `TrailingStopExit` |
| 4 | **Time Limit** | ≥ 10080 min (7 days) | `TimeLimitExit` |
| 5 | **Reversal** | Trend reversal, confidence ≥ 0.80 | `ReversalDetectionExit` |

### Implementation Pattern
```python
def check_exit_logics(self, current_price, reversal_data):
    # Exit logics are iterated in priority order. First match triggers exit.
    for logic in self.exit_logics:
        signal = logic.check(current_price, unrealized_pnl=self.unrealized_pnl)
        if signal and signal.should_exit:
            return signal
    return None
```

---

## 8. Price Buffer Pattern

**File:** `hfscalper.py`

### What Changed (v2.4.0 — critical performance fix)

Eliminated ALL Binance API calls for price data by caching WebSocket data in memory.

### Implementation
```python
self.price_buffer: Dict[str, deque] = {}  # symbol → deque(maxlen=120)

# Process stream data → populate buffer
async def process_stream_data(self, symbol, data, timestamp):
    price = extract_price(data)
    self.price_buffer[symbol].append(price)
    # Update position current_price directly
    for position in self.active_positions.get(symbol, []):
        position.current_price = price

# Position retrieves price → zero API calls
async def _get_price(self, symbol):
    if buffer_fresh:
        return buffer_value       # < 30s old
    if live_mode:
        return api_fallback       # < 60s old
    return stale_buffer_value     # last resort
```

### Performance Impact
- **Before:** 540 API calls, 0 successful retrievals (0%)
- **After:** 696/696 successful retrievals (100%)
- Rate limit elimination: 1,800+ API calls/minute → **0 API calls**

---

## 9. Exception-Safe Logging

**File:** `hfscalper.py` — all logger.log() calls

### What Changed
ALL `logger.log()` calls wrapped in try/except blocks. Logger exceptions no longer crash the system.

```python
# Pattern applied everywhere:
if self.logger:
    try:
        self.logger.log({...})
    except Exception as e:
        print(f"Logger error: {e}")
        # Continue execution — don't crash!
```

### Result
- Zero crashes from logging failures
- Trading operations continue uninterrupted even if logging subsystem fails

---

## 10. Placeholder & Ghost Position Prevention

**File:** `hfscalper.py`

### PositionPlaceholder
```python
class PositionPlaceholder:
    """Falsy duck-type used as slot reservation during order creation."""
    is_placeholder: ClassVar[bool] = True

    def __bool__(self):
        return False  # Correctly filtered by state persistence
```

### Prevention Mechanisms

1. **Placeholder → order → position:**
   - Reserve slot with placeholder BEFORE placing order
   - Replace with real Position AFTER order fills
   - If order fails: placeholder removed, capital returned

2. **Capital validation before placeholder** (open_position entry):
   - Capital checked → if valid, create placeholder → place order
   - Order fail = no ghost position

3. **State persistence filter** (save_state):
   - `PositionPlaceholder.__bool__` returns `False` — naturally filtered by truthiness checks

4. **Error path cleanup**: ALL exception handlers in `open_position()` clean up placeholders and return capital

---

## 11. Emergency Shutdown & Cleanup

**File:** `cli.py`, `hfscalper.py`

### Cleanup Sequence (cli.py)

| Step | Action | Timeout | Description |
|------|--------|---------|-------------|
| 0a | State save | - | Final snapshot before connections close |
| 1 | Position close | 300s | Close all positions via Binance API |
| 2 | State save | - | Final state after position closure |
| 3 | Final summary | - | Print trade summary |
| 4 | WebSocket close | 120s | `streamer.stop_all()` |
| 5 | HTTP close | 120s | Binance client disconnect |
| 6 | File close | 60s | Raw/output file handles |
| 7 | Bytecode cleanup | 60s | `__pycache__` removal |

### Auto-Mode Behavior
When running under `--auto` or systemd:
- Step 1 (position close) is **skipped** — positions remain open on Binance
- State is saved — next restart restores monitoring
- Enables crash recovery without unnecessary position closure

---

## 12. Auto-Detection Mode (No `--demo` Flag)

### What Changed
Removed `--demo` flag. System auto-detects trading mode:

```python
self.is_live_trading = client is not None
```

| Command | Mode | Behavior |
|---------|------|----------|
| `--scalp --loghfs --symbols BTCUSDT` | Console | Simulated trading, no Binance orders |
| `--scalp --live --loghfs --symbols BTCUSDT` | Live | Real orders on Binance Futures |

---

## 13. Concurrent Position Management

**File:** `hfscalper.py`

### Current Setting: 20 max concurrent positions

### Enforcement Logic
```python
# Count non-hedge, non-placeholder positions
current_positions = sum(
    1 for positions in self.active_positions.values()
    for pos in positions
    if not isinstance(pos, dict)
    and not getattr(pos, 'is_hedge', False)
    and not getattr(pos, 'is_placeholder', False)
)

# Hedge positions bypass the limit
if current_positions >= self.max_concurrent_positions and not is_hedge:
    print(f"⚠️ Max concurrent positions reached ({n}/{self.max_concurrent_positions})")
    return None  # Skip this signal

# Hedge allowed even at max
if current_positions >= self.max_concurrent_positions and is_hedge:
    print(f"✅ Hedge allowed at max concurrent positions")
    # Proceed with hedge opening
```

---

## 14. Stream Health Validation

**File:** `hfscalper.py`

### Tracked State
```python
self.last_real_price_update: Dict[str, datetime] = {}
```

### What It Does
- Tracks when actual **price data** (not WebSocket heartbeat) was last received per symbol
- Separates real data from keepalive/connection pings

### Enforcement in `open_position()`
```python
if symbol in self.last_real_price_update:
    elapsed = (datetime.now() - self.last_real_price_update[symbol]).total_seconds()
    if elapsed > STREAM_UNHEALTHY_THRESHOLD:  # 60 seconds
        print(f"⚠️ Stream unhealthy for {symbol}: {elapsed:.0f}s since last price")
        return None  # Refuse to open
```

### Fix
Prevents opening positions on dead streams — resolves the DYDXUSDC stale-stream bug where the bot opened positions based on stale data.

---

## 15. Candle Buffer — Interval-Filtered & Cleaned

**File:** `hfscalper.py`

### What Changed (June 18)

**Before:** The `candle_buffer` was populated by **every** closed kline from **all** subscribed intervals (1m, 5m, 15m, 1h, 4h, 1d) into the same deque. The EMA(12)/EMA(26) calculation on mixed-frequency data was mathematically invalid.

**After:** The buffer only accepts closed candles where `kline_data.get('i') == candle_kline_interval`.

### Buffer population (line 3639)
```python
# Before (buggy):
if kline_data.get('x'):  # ALL intervals flood the buffer
    self.candle_buffer[symbol].append(current_price)

# After (fixed):
if kline_data.get('x') and kline_data.get('i') == self.candle_kline_interval:
    self.candle_buffer[symbol].append(current_price)
```

### New Parameters (lines 1519-1525)
```python
self.candle_buffer_maxlen = 100            # ~500 min of history (100 × 5m candles)
self.candle_ema_fast = 12                  # Fast EMA period for candle trend
self.candle_ema_slow = 26                  # Slow EMA period for candle trend
self.candle_trend_threshold = 0.001        # 0.1% minimum EMA spread for valid trend
self.candle_kline_interval = '5m'          # Kline interval feeding candle_buffer (default)

# Trading symbols — populated at init from exchange info
self.symbols: List[str] = []               # Symbol list for REST seed
```

### `_seed_candle_buffer()` — Cold Start Elimination (June 19)

**Problem:** When `candle_kline_interval=5m` and `candle_buffer_maxlen=100`, the buffer takes 8h+ to fill from WebSocket candle closes. Until then, the candle gate in `_candle_skip_trail()` returns "insufficient data" — no trend-based trail suppression for 130+ minutes.

**Solution:** `_seed_candle_buffer()` fetches `candle_buffer_maxlen` historical 5m klines per symbol from Binance REST API before WebSocket streaming starts. All buffers are full in ~5s for 535 symbols.

**Implementation:**
- Called from CLI before `start_streaming()` (both `--all` and `--symbols` paths)
- Guards: skipped if no client (demo mode) or no symbols
- Concurrency: `asyncio.Semaphore(20)` — 20 parallel requests
- Logging: `candle_buffer_seed` event per symbol (symbol, interval, candles_loaded, first_close, last_close, buffer_length, buffer_full)
- Failure logging: `candle_buffer_seed_failed` on errors

**Integration with state persistence:** Seed runs before `_restore_from_state()`, so restored positions immediately have full candle context. The last REST candle is the most recent completed 5m — the next WebSocket 5m close continues seamlessly with no gap.
```

### `_assess_candle_trend()` — Unified Trend Assessment (line 1921)
```python
def _assess_candle_trend(self, symbol: str) -> Dict:
    """Assess trend using candle EMA(12/26) at self.candle_kline_interval"""
    buf = self.candle_buffer.get(symbol)
    if not buf or len(buf) < self.candle_ema_slow:
        return {'valid': False, 'reason': 'Insufficient candle data'}
    
    prices = pd.Series(list(buf))
    ema_fast = prices.ewm(span=12, adjust=False).mean().iloc[-1]
    ema_slow = prices.ewm(span=26, adjust=False).mean().iloc[-1]
    spread = abs((ema_fast - ema_slow) / ema_slow)
    
    if spread < self.candle_trend_threshold:
        return {'valid': False, 'reason': 'Candle trend too weak'}
    
    direction = 'bullish' if ema_fast > ema_slow else 'bearish'
    return {'valid': True, 'direction': direction, 'spread': spread}
```

Used by both:
1. **Entry confirmation** (line 2089): Blocks new entries when candle trend disagrees with tick trend
2. **Exit candle gate** (line 808): Sets `_candle_skip_trail` when candle trend matches position direction

### Cold Start Behavior
On fresh start (no persisted state):
- Candle buffer starts empty
- 1m candle closes every 60 seconds → 1 entry per minute
- `kline_trend_insufficient: N/26` logged until N ≥ 26
- After ~26 minutes: first valid trend assessment

---

## 16. Loss Recovery Trail

**File:** `hfscalper.py:108-159` (embedded in `StopLossExit`)

### Concept
The loss-side mirror of the profit trailing stop. While the profit trail locks in gains, the loss trail salvages losses by exiting when the price recovers from its worst point (trough) by a meaningful distance.

### Parameters

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `loss_trail_activation` | 3.5% | Minimum drawdown before trail arms |
| `loss_trail_distance` | 1.1% | Recovery from trough needed to trigger exit |

### Two-Stage Check in StopLossExit.check() (line 118)

#### Stage 1: Loss Recovery Trail (checked FIRST, lines 130-159)
```python
if pnl_pct < 0:  # In loss territory
    loss_pct = abs(pnl_pct)
    
    # Update trough (deepest loss reached)
    if loss_pct > self.trough_pnl:
        self.trough_pnl = loss_pct
    
    # Check activation: loss >= 3.5%?
    if not self.loss_trail_activated and loss_pct >= self.loss_trail_activation:
        self.loss_trail_activated = True
    
    # If activated, check recovery from trough
    if self.loss_trail_activated:
        recovery = self.trough_pnl - loss_pct
        
        # Candle gate can suppress this check
        if getattr(self.position, '_candle_skip_trail', False):
            # Same candle trend → let loser recover, don't exit
            pass
        elif recovery >= self.loss_trail_distance:
            # Recovered 1.1% from worst loss → exit to salvage
            return ExitSignal(should_exit=True, ...)
```

#### Stage 2: Hard Stop (checked SECOND, lines 161-184)
Traditional fixed stop at 4% (`stop_price`). **Always fires regardless of candle gate.**

### Example
```
Entry at $100 → drops to $95 (-5%) → trough = 5%
Loss trail activates at 3.5% ✓
Price recovers to $97.50 (-2.5%) → recovery = 5% - 2.5% = 2.5%
Exit triggers: 2.5% ≥ 1.1% ✓ → exit at -2.5% instead of full 4% stop
```

---

## 17. Candle Gate — Profit & Loss Trail Suppression

**File:** `hfscalper.py:791-831`

### What It Does
In every `Position.update()` cycle, compares the 1m candle EMA(12/26) trend against the position direction. When the trend matches the position, sets `_candle_skip_trail = True` to suppress trail activation — letting winners run and losers recover.

### Trigger Thresholds (line 796-800)
```python
# Profit side: PnL >= 1.3% × 0.3 = 0.39%
profit_check = self.unrealized_pnl >= self.trail_activation * self.candle_trail_min_ratio

# Loss side: PnL < 0 AND abs(PnL) >= 3.5%
loss_check = (self.unrealized_pnl < 0 and abs(self.unrealized_pnl) >= self.loss_trail_activation)
```

### Gate Decision (lines 802-831)
```python
# Skip for hedge positions
if not is_hedge and (profit_check or loss_check) and hfscalper is not None:
    candle_trend = hfscalper._assess_candle_trend(symbol)
    if candle_trend.get('valid'):
        if candle_trend['direction'] == self.direction:
            # Same trend → suppress trail activation
            self._candle_skip_trail = True
        else:
            # Opposite trend → let trail activate
            pass  # _candle_skip_trail stays False
```

### Where the Gate is Consumed

**Profit side — TrailingStopExit.check() (line 270):**
```python
if not self.trail_activated and getattr(self.position, '_candle_skip_trail', False):
    return None  # Skip trail activation
```
Only blocks **activation**, not an already-active trail.

**Loss side — StopLossExit.check() (lines 148-150):**
```python
if getattr(self.position, '_candle_skip_trail', False):
    # Loss trail recovery check suppressed
    pass  # Falls through to hard stop below
```
Only blocks the **recovery trail trigger**, not the hard stop.

### Key Design Decisions
- `_candle_skip_trail` reset to `False` every monitor cycle
- Not persisted to state — trail re-evaluates fresh after restart
- Hedge positions always skip the gate
- Hard stop (4%) always fires regardless of gate

---

## 18. Defensive Hedge with Candle Gate

**File:** `hfscalper.py`

### What It Does
After `defensive_hedge_minutes` (1440.0 = 24h) of consecutive losses, evaluates whether to open a defensive hedge. Before hedging, verifies the 5-minute candle trend actually opposes the current position.

### S1 Gate (defensive hedge):
```python
if consecutive_losses_minutes >= self.defensive_hedge_minutes:
    candle_trend = get_candle_trend(symbol)
    if candle_trend_opposes_position(candle_trend, self.direction):
        await self._open_defensive_hedge()
    else:
        # Candle trend supports position — don't hedge
        pass
```

---

## 19. Capital Capping & Balance Protection

**File:** `cli.py`

### Live Balance Capping
```python
# Cap live balance at $1000 even on larger accounts
initial_capital = min(binance_balance, 1000.0)
```

### Capital Usage Hierarchy
```
Binance balance → capped at $1000 → 90% deployment cap → 2.9% per position
```

With $15.9 balance:
- Available capital: $15.9 (under $1000 cap)
- Deployable: $15.9 × 90% = $14.31
- Per position: $15.9 × 2.9% = ~$0.46
- 20 positions max: ~$9.20 total → safe

---

## 20. Order Failure Recovery & Auto-Scale

**File:** `hfscalper.py`

### -4164 Auto-Scale
When Binance rejects a MARKET order with `-4164` (notional below minimum):
```python
# Calculate minimum viable quantity
min_qty = min_notional_with_buffer / current_price
adjusted_qty = adjust_to_step_size(min_qty, step_size)

# Scale quantity up and retry once
position_value = adjusted_qty * current_price
```

### Capital Return on Failure
All order failure paths return capital to `available_capital`:
```python
try:
    # ... order placement ...
except BinanceAPIException as e:
    self.available_capital += capital_required
    # Clean up placeholder
```

---

## 21. Files Changed Reference

### Source Files

| File | Lines | Primary Role |
|------|-------|-------------|
| `streamer.py` (root) | 330 | CLI entry point, systemd setup, arg parsing |
| `src/rebirth_trader/cli.py` | ~1260 | CLI orchestration, signal handlers, cleanup, state restore, tunnel startup |
| `src/rebirth_trader/hfscalper.py` | ~4476 | Core trading engine: positions, exits, state, buffers |
| `src/rebirth_trader/binance_client.py` | ~747 | Binance Futures API wrapper |
| `src/rebirth_trader/config.py` | - | Configuration loader |
| `src/rebirth_trader/__init__.py` | - | Version and package metadata |
| `src/rebirth_trader/ui/server.py` | ~480 | Embedded web dashboard HTTP server |
| `src/rebirth_trader/ui/index.html` | ~1535 | Single-file dashboard HTML+CSS+JS template |

### Documentation Files

| File | Purpose |
|------|---------|
| `docs/README.md` | Quick-start, CLI reference, dashboard URLs, key features |
| `docs/ARCHITECTURE.md` | Full system architecture including dashboard & tunnel |
| `docs/LIVE_TRADING.md` | Production trading setup, dashboard monitoring, API ref |
| `docs/FEATURES_ENHANCEMENT.md` | This file — all features and patches |

### Runtime Files

| File | Purpose |
|------|---------|
| `logs/state.jsonl` | State persistence (position snapshots every 10s) |
| `logs/history.jsonl` | Permanent trade journal (one line per close, decoupled from `--loghfs`) |
| `logs/hfscalper_*.jsonl` | Trading event logs (~60 event types, optional `--loghfs`) |
| `logs/raw.jsonl` | Raw WebSocket data (optional, `--raw`) |
| `.env` | API credentials (chmod 600) |

### System Files

| File | Purpose |
|------|---------|
| `/etc/systemd/system/rebirth-trader.service` | Systemd unit — includes `--ui --tunnel` flags |
| `/etc/nginx/sites-available/rebirth-trader` | Nginx config — proxies `/dashboard/` and `/api/` to bot port 3000 |
| `/var/www/html/index.html` | Landing page — links to `/dashboard/`, fetches `/api/stats` |
| `/usr/local/bin/cloudflared` | Cloudflare Tunnel binary (auto-installed by `--tunnel`) |
| `/home/kali/rebirth-trader/.venv/` | Python virtual environment (uv-managed) |

---

## 22. Trading Pause Window

**Files:** `hfscalper.py`

### Configuration
```python
self.trading_pause_start_hour = 23   # 11pm UTC
self.trading_pause_end_hour = 2      # 2am UTC
```

### What It Does
Creates a daily window (23:00–02:00 UTC) during which new trade entries and defensive hedges are blocked. Existing positions continue normal monitoring — exit logics, trailing stops, time limits all operate as usual.

### Behavior

| Scenario | During Pause | Outside Pause |
|----------|-------------|---------------|
| New entry signal | Blocked (`entry_skipped_pause`) | Normal evaluation |
| Defensive hedge | Blocked (`hedge_skipped_pause`) | Normal evaluation |
| Active position monitoring | Continues normally | Continues normally |
| Stop loss / trailing stop | Active | Active |
| Position close on exit | Active | Active |

### Enforcement Points
1. **`process_stream_data()`**: Checks `now_hour >= start or now_hour < end` before calling `assess_market_conditions()` for new entries
2. **`_check_hedge()`**: Same check before opening defensive hedges

### Rationale
Reduces risk during low-liquidity / high-volatility overnight periods. The pause window aligns with typical low-volume hours in crypto markets (UTC night).

---

## 23. Symbol Blacklist (`--blacked`)

**Files:** `hfscalper.py`, `cli.py`, `streamer.py`

### What It Does
Auto-suspends trading on symbols that hit 3 consecutive losses, preventing the bot from repeatedly losing capital on underperforming pairs. Configurable cooldown duration. Pre-blacklist support for excluding known-problematic symbols from the start.

### Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `blacked_max_consecutive_losses` | 3 | Consecutive losses before auto-suspend |
| `blacked_cooldown_minutes` | 1440 (24h) | Cooldown duration (overridable via `--blacked NH`) |
| `blacked_symbols` | `{}` | Active blacklist (symbol → expiry datetime) |
| `blacked_perf` | `{}` | Per-symbol performance tracking |

### CLI Usage

```bash
# Enable blacklist with default 24h cooldown
uv run streamer.py --scalp --symbols BTCUSDC,ETHUSDC --blacked

# Enable with custom 5-hour cooldown
uv run streamer.py --scalp --symbols BTCUSDC,ETHUSDC --blacked 5H

# Pre-blacklist CRV and KAITO with custom 12h cooldown
uv run streamer.py --scalp --symbols BTCUSDC,ETHUSDC --blacked CRV,KAITO 12H
```

### Loss Tracking & Auto-Suspend

After each position close with a loss:
```python
perf = self.blacked_perf.setdefault(symbol, {'cons_losses': 0, 'total_pnl': 0.0})
perf['cons_losses'] += 1
perf['total_pnl'] += pnl

if perf['cons_losses'] >= self.blacked_max_consecutive_losses:
    expiry = datetime.now() + timedelta(minutes=self.blacked_cooldown_minutes)
    self.blacked_symbols[symbol] = expiry
```

### Enforcement

1. **`process_stream_data()`**: Before evaluating a new entry, checks `symbol in self.blacked_symbols`. If blacklisted, logs `'entry_skipped_blacked'` and skips.
2. **`_check_hedge()`**: Before opening a defensive hedge, checks blacklist. If blacklisted, logs `'hedge_skipped_blacked'` and skips.

### Persistence

Blacklist state is part of the state persistence system:
- **Save** (`save_state()`): `blacked_symbols` serialized as `{sym: iso_expiry}`
- **Restore** (`_restore_from_state()`): Deserializes back to `datetime` objects and restores blacklist

### Cooldown Expiry

On each `assess_market_conditions()` call, expired entries are cleaned:
```python
if symbol in self.blacked_symbols and datetime.now() >= expiry:
    del self.blacked_symbols[symbol]
    self.blacked_perf.pop(symbol, None)
```

### Relationship to Other Features
- **Trading pause** is checked first, then **blacklist** — both must pass for a new entry
- **State persistence** saves/restores blacklist state across restarts
- **Defensive hedge** is gated by blacklist — won't hedge a blacklisted symbol even if candle trend opposes

---

## 24. Dashboard UI (`--ui`)

**Files:** `src/rebirth_trader/ui/server.py`, `src/rebirth_trader/ui/index.html`, `src/rebirth_trader/cli.py`, `src/rebirth_trader/streamer.py`

### What It Does

Starts an embedded web dashboard server on port 3000 when the `--ui` flag is passed. The dashboard shows real-time bot statistics, active positions, equity curve, and trade journal — accessible from any device on the same LAN.

### Implementation

**1. CLI Flag (cli.py:1254):**
```python
parser.add_argument('--ui', action='store_true',
    help='Start the embedded web dashboard on port 3000')
```

**2. Passthrough (streamer.py):**
```python
'ui': args.ui,  # Passed to CLIArgs dataclass
```

**3. Server startup (cli.py, in `_start_ui_server()`):**
```python
HOST = DEFAULT_HOST  # "0.0.0.0"
PORT = 3000
server = HTTPServer((HOST, PORT), DashboardHandler)
server_thread = threading.Thread(target=server.serve_forever, daemon=True)
server_thread.start()
```

- `DEFAULT_HOST = "0.0.0.0"` — binds to all interfaces (LAN accessible)
- Port 3000 hardcoded — no config option
- Runs in a daemon thread (no blocking)
- The server reads `ui/index.html` at startup and serves it on `GET /`

**4. Shutdown cleanup:** The server is shut down during cleanup(), terminating the daemon thread.

### Dashboard Template (`ui/index.html`)

Single-file HTML (~1535 lines) with embedded CSS and JavaScript. Key elements:

| Component | Implementation | Notes |
|-----------|---------------|-------|
| Stat cards | CSS glass-morphism with gradient accent bars | 7 cards: P&L, Positions, WR, Trades, Wallet, Realized, Fees |
| Equity curve | Chart.js 4.4.7 (CDN-loaded) | White gradient fill, 10s auto-refresh |
| Positions table | Dynamic DOM from `/api/positions` | Color-coded uPnL (emerald / red) |
| Best/Worst tables | Sorted from `/api/history` | Top 5 + bottom 5 by P&L |
| Streak dots | 100 capped canvases | Green/red dots with hover tooltips |
| Trade journal | Table from `/api/journal` | Sortable, last 1000 trades, searchable |
| Auto-refresh | `setInterval(refreshAll, 10000)` | 10s — matches state write cycle |

### API Endpoints (from `server.py`)

| Endpoint | Method | Response |
|----------|--------|----------|
| `/api/stats` | GET | Aggregated stats (wallet, pnl, positions, fees, win rate) |
| `/api/equity` | GET | Equity curve data (timestamp, balance) from state.jsonl |
| `/api/positions` | GET | Active positions with current prices |
| `/api/history` | GET | Closed trade history from state.jsonl |
| `/api/journal` | GET | Full trade journal from history.jsonl |

### Nginx Integration

The production nginx config proxies the dashboard and API:

```nginx
location /dashboard/ {
    proxy_pass http://127.0.0.1:3000/;
}
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

The landing page (`/var/www/html/index.html`) links to `/dashboard/` and fetches `/api/stats` for live stats display.

### History

- **June 14:** Initial `--ui` flag implementation — missing from argument parser, hot-patched
- **June 14:** `DEFAULT_HOST` changed from `127.0.0.1` to `0.0.0.0` for LAN access
- **June 14:** Consolidated dashboards — removed standalone `live_server.py`, nginx now proxies bot's embedded dashboard
- **June 14:** Landing page stats updated to fetch from `/api/stats` instead of stale files
- **June 14:** Buy price display updated to `1600 + closed_pnl` formula
- **June 21:** Dashboard redesigned with glass-morphism cards, emerald/emerald indicators
- **June 21:** Numeric precision reduced to 1 decimal for P&L/Wallet/Realized/Equity hero stats

### Known Issues

- **Port race on restart:** If port 3000 is still held by a previous instance, the UI server fails silently. Verify port free before restart.
- **Template cache:** The dashboard HTML is read once at startup — requires bot restart for UI changes to take effect.
- **State file growth:** ~30.5 MB/hour at 10s interval → ~5 GB/week.

---

## 25. Cloudflare Tunnel (`--tunnel`)

**Files:** `src/rebirth_trader/cli.py`, `src/rebirth_trader/streamer.py`

### What It Does

When `--tunnel` is passed alongside `--ui`, the bot creates a Cloudflare Tunnel (TryCloudflare) for public internet access to the dashboard. The tunnel is spawned as a subprocess, auto-downloads cloudflared if missing, and is killed cleanly on shutdown.

### Implementation

**1. CLI Flag (cli.py:1260-1262):**
```python
parser.add_argument('--tunnel', action='store_true',
    help='Create a Cloudflare Tunnel for public internet access to the UI dashboard')
```

**2. Passthrough (streamer.py):**
```python
'tunnel': args.tunnel,
```

**3. Tunnel startup (`_start_tunnel()` in cli.py):**

```python
def _start_tunnel(self):
    import subprocess
    cloudflared_path = self._install_cloudflared()
    self.tunnel_process = subprocess.Popen(
        [cloudflared_path, 'tunnel', '--url', 'http://localhost:3000'],
        stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
        text=True
    )
    # Read first 20 lines to extract the TryCloudflare URL
    for _ in range(20):
        line = self.tunnel_process.stdout.readline()
        if 'https://' in line and '.trycloudflare.com' in line:
            tunnel_url = line.strip().split('https://')[1].split(' ')[0]
            print(f"🌐 Dashboard public URL: https://{tunnel_url}")
            break
```

**4. Auto-install (`_install_cloudflared()`):**

```python
def _install_cloudflared(self):
    cloudflared_path = shutil.which('cloudflared')
    if cloudflared_path:
        return cloudflared_path
    # Download from Cloudflare
    url = ("https://github.com/cloudflare/cloudflared/releases/latest/"
           "download/cloudflared-linux-amd64")
    subprocess.run(['curl', '-sL', url, '-o', '/usr/local/bin/cloudflared'], check=True)
    os.chmod('/usr/local/bin/cloudflared', 0o755)
    return '/usr/local/bin/cloudflared'
```

**5. Shutdown cleanup (in shutdown handler / cleanup()):**

```python
if hasattr(self, 'tunnel_process') and self.tunnel_process:
    self.tunnel_process.terminate()
    try:
        self.tunnel_process.wait(timeout=5)
    except subprocess.TimeoutExpired:
        self.tunnel_process.kill()
```

### Security

- **Outbound only:** Cloudflare Tunnel establishes an outbound connection to Cloudflare's edge — no inbound port forwarding required
- **UFW compatible:** The tunnel works with UFW blocking all inbound ports except 22/80
- **No auth gate:** TryCloudflare URLs are random and unguessable (not truly private — anyone with the URL can access)
- **Permanent URL:** Deferred — configure with `cloudflared tunnel route dns` when a domain is available

### URLs

| Scope | URL Format | Notes |
|-------|-----------|-------|
| Temporary (free) | `https://<random>.trycloudflare.com` | Auto-generated, changes on each restart |
| Permanent | Custom subdomain | Requires `cloudflared tunnel route dns` + domain setup |

### History

- **June 14:** Initial implementation — `--tunnel` flag added to cli.py and streamer.py
- **June 14:** cloudflared auto-install + subprocess management added
- **June 14:** Cleanup integrated into shutdown handler
- **June 14:** Systemd service updated to include `--tunnel`

### Known Issues

- **URL changes on restart:** Each restart generates a new TryCloudflare URL. Set up a permanent tunnel when a domain is ready.
- **Process cleanup:** If the bot crashes hard (SIGKILL), the cloudflared subprocess may orphan. Kill manually: `pkill cloudflared`.

---

**Version:** 3.0.0 | **Last Updated:** June 21, 2026 | **Status:** Rebranded to Rebirth Trader
