# Rebirth Trader — System Architecture Reference

**Version 3.0.0** | **Production Ready** | **June 21, 2026**

Complete architectural reference for the Rebirth Trader Binance Futures scalping system: initialization, data flow, position lifecycle, exit logic, state persistence, shutdown sequence, and error handling.

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
9. [Exit Logic System](#9-exit-logic-system)
10. [Candle Gate — Profit & Loss Trail Suppression](#10-candle-gate--profit--loss-trail-suppression)
11. [Hedge Management](#11-hedge-management)
12. [Trading Pause Window](#12-trading-pause-window)
13. [Symbol Blacklist](#13-symbol-blacklist)
14. [Dashboard & Monitoring](#14-dashboard--monitoring)
15. [State Persistence](#15-state-persistence)
16. [Signal Handler & Shutdown Sequence](#16-signal-handler--shutdown-sequence)
17. [Auto-Mode & Systemd Integration](#17-auto-mode--systemd-integration)
18. [Error Handling & Recovery](#18-error-handling--recovery)

---

## 1. System Overview & Data Flow

### High-Level Architecture

The system connects to Binance Futures (Hedge Mode) via WebSocket streams. A streamer module passes CLI arguments to the CLI handler, which instantiates the core trading engine. The engine manages per-position monitoring coroutines (with a configurable maximum) and an optional embedded UI server for real-time dashboard access.

```
Binance Futures (Hedge Mode)
    ↕ WebSocket streams
Streamer Module
    ↓ CLI arguments parsed
CLI Handler
    ↓ Core trading engine instance
Trading Engine
    ↓ per-position coroutines         ↓ UI mode flag
Position Monitor × N (configurable)   UI Server (LAN accessible, configurable port)
    ↓ exit logic loop                 ↓ Reverse proxy
Binance API (position close)          Public URL (via tunnel)
```

### Data Pipeline

```
WebSocket data (kline/ticker/aggTrade)
    → Parse and validate price data
    → In-memory price buffer (rolling window per symbol)
    → In-memory candle buffer (filtered closed candles only)
    → Periodic market assessment
        → Trend detection via EMA crossover
        → RSI filter
        → Volatility check
        → Confidence scoring
        → Candle trend confirmation at configured interval
    → Position sizing and capital validation
    → Order placement on Binance
    → Independent monitoring coroutine per position
```

---

## 2. Core Design Principles

### Position-Centric Architecture
- Each position runs in its own `asyncio` coroutine with independent error isolation
- Failures in one position never cascade to others
- All positions share resources via a callback pattern
- Configurable maximum concurrent positions (hedges excluded from limit)

### Price Buffer Pattern
- WebSocket price data cached in memory — zero Binance API calls for monitoring prices
- Per-symbol price buffer (rolling deque with configurable max length) populated by the stream processing function
- Positions retrieve prices via callbacks — buffer first, API fallback on staleness
- Configurable staleness and unhealthy stream thresholds

### Candle Buffer Pattern
- Separate buffer for trend confirmation only (not price retrieval)
- Per-symbol candle buffer (rolling deque with configurable max length) populated by closed candles at the configured interval (REST-seeded at startup)
- Filtered by candle kline interval — only candles matching this interval are appended
- Cold start eliminated: the candle seeder fetches historical candles from Binance REST API per symbol before streaming begins, using controlled concurrency for efficient handling of large symbol sets
- Used by both exit candle gate and entry candle confirmation

### No Demo Flag
- Trading mode auto-detected: live trading is active when a Binance client is provided
- No live flag = console mode (simulated trading, no Binance orders)
- Live flag = real orders on Binance Futures

### Singleton Configuration
- All risk parameters defined once in the trading engine's initialization
- Propagated to positions via callbacks and dataclass defaults
- Single change point for all configurable parameters

---

## 3. Entry Flow & Initialization

### Call Chain

```
Entry point (main)
    → Parse CLI arguments (including persistence setup)
    → CLI arguments dataclass wrapper
    → CLI handler run method
        → Set up signal handlers for graceful shutdown
        → Display startup banner
        → Check for state restoration
        → Initialize the trading engine
        → Restore from persisted state if applicable
        → Seed candle buffers from REST API (before WebSocket streaming)
        → Begin WebSocket streaming
        → Main loop (until signal received)
            → Periodic state saves
        → Cleanup and shutdown sequence
```

### Trading Engine Initialization

The trading engine's initializer performs the following key operations:

1. **Trading mode detection:** Determines whether live trading or simulation mode is active based on client availability.

2. **Risk parameter configuration:** Sets configurable parameters including fee rate, minimum notional threshold, risk per trade, maximum risk adjustment, capital limits per position and in total, stop loss threshold, profit target, trail activation thresholds, trail distances for both profit and loss sides, and maximum holding duration.

3. **Leverage tiers:** Configures leverage levels mapped to confidence tiers (high, medium, low).

4. **Trading parameters:** Configures maximum concurrent positions, trend detection threshold, volatility threshold, confidence normalization factor, minimum confidence threshold, and signal check interval.

5. **EMA and buffer parameters:** Configures slow and fast EMA periods, buffer size thresholds, price buffer and candle buffer maximum lengths, candle EMA parameters, trend threshold, and candle kline interval.

6. **Trading pause window:** Defines hours during which new entries and hedges are blocked.

7. **Symbol blacklist:** Configures consecutive loss threshold and cooldown duration for auto-suspending underperforming symbols.

8. **Concurrency control:** Initializes sets and locks for coordinating position openings and preventing race conditions.

9. **Stream health tracking:** Initializes per-symbol tracking of the last real price update timestamp.

10. **Persistence startup:** If persistence mode is enabled, begins periodic state saving.

---

## 4. CLI Flags Reference

| Flag | Type | Description |
|------|------|-------------|
| `--scalp` | store_true | Enable trading engine mode |
| `--live` | store_true | Real money trading on Binance Futures |
| `--persist` | store_true | Save/restore state and install systemd service |
| `--auto` / `-a` | store_true | Bypass all confirmations (for automation) |
| `--loghfs` | store_true | Enable structured event logging |
| `--ui` | store_true | Start embedded web dashboard on configurable port |
| `--tunnel` | store_true | Create Cloudflare Tunnel for public internet access (requires `--ui`) |
| `--symbols` | str | Comma-separated symbol list |
| `--all` | store_true | All tradable futures symbols |
| `--raw` | store_true | Save raw WebSocket data |
| `--save` | str | Save stream to file |
| `--blacked` | str[*] | Symbol blacklist: repeated consecutive losses trigger auto-suspend with configurable cooldown. Supports pre-blacklisting symbols and overriding cooldown duration. Persisted across restarts. |
| `--version` | - | Show version number and exit |
| `-h, --help` | - | Show help message and exit |

**Mutual exclusion:** `--symbols` and `--all` are mutually exclusive. Exactly one is required.

---

## 5. Stream Data Processing & Price/Candle Buffers

### Stream Processing

Called by the streamer module for every kline, ticker, and trade message.

**Flow:**
1. Extract price from ticker close, trade price, or kline close
2. Reject invalid prices
3. Update the last real price update timestamp
4. Populate the per-symbol price buffer (rolling deque with configurable max length)
5. For kline data: populate the candle buffer only when the kline's interval matches the configured candle interval AND the candle is closed
6. Update position current price directly (zero-latency feed)
7. Calculate rolling volatility from price returns
8. Periodically call the market assessment function

**Candle buffer filter:** Only closed candles at the configured interval populate the buffer — no mixing from other intervals.

**Cold start elimination:** Before WebSocket streaming begins, the candle seeder fetches historical candles per symbol via Binance REST API using controlled concurrency. All candle buffers are populated before the first WebSocket candle close arrives, making EMA calculations immediately available.

### Price Validation
- Null/zero prices are rejected — prevents NaN propagation
- Last real price update is tracked separately from WebSocket heartbeat messages
- Prevents opening positions on dead streams

### Price Retrieval

Uses a three-tier fallback strategy:
1. Check price buffer for recent data (within configurable staleness threshold) — return it
2. If data is stale or live mode: API fallback via Binance
3. If no data at all: return None

All thresholds (API timeout, buffer staleness, stream unhealthiness) are configurable.

---

## 6. Signal Generation & Market Assessment

### Market Assessment

Called periodically per symbol after the price buffer contains sufficient entries.

**Prerequisites:**
- Price buffer must contain enough entries for EMA calculation

**Assessment steps:**

1. **Trend detection** — EMA crossover analysis:
   - Calculates fast and slow EMAs from the price buffer
   - Determines trend direction (bullish or bearish)
   - Evaluates trend strength and validity against a configurable threshold

2. **RSI filter:**
   - Identifies overbought and oversold conditions
   - Extreme RSI values can override EMA trend signals

3. **Volatility filter:**
   - Skips entry if volatility exceeds a configurable threshold

4. **Confidence scoring:**
   - Normalizes trend strength into a confidence score (0.0–1.0)

5. **Leverage selection:**
   - Maps confidence and volatility levels to predefined leverage tiers

6. **Direction and entry signal:**
   - Determines trade direction based on trend and RSI conditions
   - Entry is permitted when trend is valid and RSI is not overextended

7. **Candle confirmation:**
   - Verifies that the candle EMA trend at the configured interval agrees with the tick trend before allowing entry
   - If candle trend disagrees, entry is blocked
   - If insufficient candle data exists, entry is allowed as a cold-start grace

---

## 7. Position Lifecycle

### Lifecycle States

```
IDLE → PLACEHOLDER → OPENING (order pending) → ACTIVE (monitoring) → CLOSED
```

### Detailed Flow: Position Opening

#### Step 1: Pre-checks
- **Stream health:** Last real price update must be sufficiently recent
- **Duplicate gate:** Checks for duplicate entries in the opening set
- **Position limit:** Counts active positions, skips if at capacity (hedges exempt)
- **Capital check:** Confirms sufficient available capital

#### Step 2: Placeholder Reservation
A placeholder object is appended to the active positions list. The placeholder is correctly handled by state persistence filters and position-count logic (it evaluates as falsy).

#### Step 3: Order Placement
- Set leverage on Binance
- Set margin type
- Place MARKET order via Binance API
- Handle auto-scale errors: if order fails due to minimum notional, rescale quantity and retry once
- Wait for fill confirmation with configurable timeout

#### Step 4: Position Creation
A Position object is created with all relevant parameters including symbol, direction, entry and exit prices, quantity, capital used, unique position ID, hedge status, and all exit-related parameters and timestamps.

#### Step 5: Placeholder Replacement
Under a position lock, the placeholder is replaced at the same list index with the real Position object.

#### Step 6: Callback Attachment
Callbacks for price retrieval, position closing, reversal detection, and hedge checking are attached. The logger and trading engine reference are also provided.

#### Step 7: Monitor Coroutine Start
An independent asyncio task is created for the position's monitoring loop.

### Monitor Loop

Runs as an independent coroutine until the position is marked as closed.

```
while position is not closed:
    1. Get current price (buffer → API fallback → stale fallback)
    2. Update unrealized PnL
    3. Detect reversal signals
    4. Check candle gate — evaluate whether trail activation should be suppressed
    5. Check exit logics in priority order
    6. If exit triggered, close position
    7. Wait before next cycle
```

**Error isolation:** Every step is wrapped in try/except — errors are logged, position continues.

---

## 8. Position Sizing & Capital Management

### Sizing Flow

The position sizing algorithm:
- Calculates adjusted risk as a configurable percentage of available capital
- Applies leverage from the appropriate confidence tier
- Computes position value from adjusted risk and leverage
- Enforces minimum notional requirements (with a configurable buffer above Binance's minimum)
- Calculates coin quantity from position value and current price
- Enforces exchange minimum quantity constraints
- Caps capital per position at a configurable percentage of total capital

### Capital Gates

| Check | Description |
|-------|-------------|
| Per-position cap | Configurable percentage cap on single position capital |
| Total usage cap | Configurable percentage cap on sum of all active positions |
| Minimum notional | Binance minimum plus configurable buffer |
| Available capital | Deducted on open, returned on close |
| Balance cap (live) | Configurable maximum, even with larger balance |

---

## 9. Exit Logic System

### Exit Signal Model

A structured exit signal carries:
- Whether to exit
- The exit price
- The exit reason description
- The logic name (e.g., stop loss, take profit, trailing stop, time limit, reversal)

### Priority Order

Exit logics are registered and checked in strict priority order:

| Priority | Logic | Trigger |
|----------|-------|---------|
| 1 | **Stop Loss + Loss Recovery Trail** | Two-stage: loss trail checks first, hard stop second |
| 2 | **Take Profit** | PnL reaches configurable profit target |
| 3 | **Trailing Stop** | Configurable profit threshold reached, then pullback triggers exit |
| 4 | **Time Limit** | Position exceeds configurable maximum holding duration |
| 5 | **Reversal** | Trend reversal signal with sufficient confidence |

### Stop Loss — Two-Stage Check

#### Stage 1: Loss Recovery Trail (checked first)
The loss-side mirror of the profit trailing stop:
- Tracks the deepest loss percentage reached
- Activates when drawdown exceeds a configurable threshold
- Once activated, measures recovery from the trough
- If recovery exceeds a configurable distance, triggers exit via Loss Trail signal
- Candle gate protection: if the candle trend still favors the position, recovery check is suppressed

#### Stage 2: Hard Stop (checked second)
A traditional fixed stop at a configurable distance from entry. Always fires regardless of candle gate.

### Profit Trailing Stop

- Tracks the maximum profit percentage reached
- Activates when profit exceeds a configurable threshold
- Once activated, measures pullback from the peak
- If pullback exceeds a configurable distance, triggers exit
- Candle gate protection: If the candle trend still favors the position and the trail is not yet activated, activation is suppressed — letting winners run

---

## 10. Candle Gate — Profit & Loss Trail Suppression

### Overview

The candle gate is a runtime check that compares the candle trend (EMA crossover at the configured interval) against the position direction. It suppresses trail activation when the candle trend still favors the position.

### Gate Activation Thresholds

- **Profit side:** Activates when PnL reaches a configurable fraction of the trail activation threshold
- **Loss side:** Activates when the loss exceeds the loss trail activation threshold

Below these thresholds, the gate is skipped entirely.

### Gate Decision Matrix

| Candle Trend vs Position | Effect on Profit Trail | Effect on Loss Trail |
|--------------------------|------------------------|---------------------|
| **Same direction** | Trailing stop cannot activate — winner runs higher | Loss recovery trail cannot trigger on recovery — loser bounces |
| **Opposite direction** | Trail activates normally — lock profit on pullback | Loss trail checks recovery from trough — salvage on bounce |
| Trend too weak / insufficient data | Same as opposite | Same as opposite |

### Scope Limits
- **Profit trail:** Gate only blocks activation of the trailing stop. Once armed, the trail operates normally regardless of gate.
- **Loss trail:** Gate blocks the recovery check. The hard stop always fires regardless.
- **Hedges:** Gate is skipped entirely for hedge positions.
- **Runtime only:** The gate flag is reset every monitor cycle and is not persisted.

---

## 11. Hedge Management

### Hedge Mode Setup
- Binance Futures account is set to Hedge Mode, supporting both LONG and SHORT positions simultaneously per symbol
- Each position tracks its position side

### Defensive Hedge Gate
When a position experiences repeated consecutive losses exceeding a configurable threshold:
1. Check if a candle trend exists for the symbol
2. If candle trend opposes the current position, open a defensive hedge
3. The hedge is the opposite direction of the current position

The hedge itself bypasses the maximum concurrent positions limit.

### Hedge Pair Linking
On state restoration, positions are linked to their hedge counterpart via a pair ID. This enables combined P&L tracking and coordinated exit decisions.

---

## 12. Trading Pause Window

### Configuration
Configurable start and end hours define a daily trading pause window.

### Behavior
During the pause window:
- **New entries blocked:** Market assessment returns early with a pause reason
- **New hedges blocked:** Defensive hedge creation is skipped during pause window
- **Existing positions unaffected:** All active monitoring coroutines continue normally — only new position creation is blocked
- **Logged:** Pause-related events are recorded

### Enforcement Points
- Stream processing at signal evaluation: checks current time against pause window
- Hedge checking: same check before opening defensive hedges

---

## 13. Symbol Blacklist

### What It Does
Auto-suspends trading on symbols that show repeated consecutive losses, preventing the bot from burning capital on consistently failing pairs.

### Configuration
Configurable parameters include the consecutive loss threshold and cooldown duration (overridable via CLI).

### CLI Usage
The feature supports enabling blacklist with default cooldown, pre-blacklisting symbols on startup, and overriding cooldown duration.

### Loss Tracking
After each position close with a loss, the bot tracks consecutive losses and total P&L per symbol. When the threshold is reached, the symbol is blacklisted for the configured cooldown duration.

### Enforcement Points
- **New entries:** Skipped with a logged event if the symbol is blacklisted
- **Defensive hedges:** Skipped with a logged event if the symbol is blacklisted

### Persistence
Blacklist state is saved and restored via the state persistence system, including symbol-to-expiry mappings.

### Cooldown Expiry
On each market assessment call, expired blacklist entries are automatically cleaned up.

---

## 14. Dashboard & Monitoring

### Architecture

```
Trading Engine (UI mode flag)
    ↓
UI Server on configurable address and port
    ├── GET /            → Dashboard HTML
    ├── GET /api/stats      → Live aggregated stats
    ├── GET /api/equity     → Equity curve data
    ├── GET /api/positions  → Active position details
    ├── GET /api/history    → Closed trade history
    └── GET /api/journal    → Raw full journal
    ↓ Reverse proxy
    ├── Dashboard route → proxied to UI server
    └── API route       → proxied to UI server
    ↓ Cloudflare tunnel
    Public URL → local UI server
```

### UI Server
- **Fast:** Pure Python HTTP server with zero external dependencies
- **Bind:** LAN accessible when UI mode is active
- **State source:** Reads state file on each request
- **Trade journal:** Reads history independently
- **Dashboard template:** Single-file HTML with embedded CSS and JavaScript

### Dashboard Template
Single-file HTML with embedded CSS and JavaScript. Features:
- **Stat cards:** Glass-morphism cards with colored accent bars
- **Equity curve:** Chart canvas with gradient fill
- **Positions table:** Live positions with P&L coloring
- **Best/Worst tables:** Top performers by P&L
- **Streak dots:** Visual trade streak history
- **Trade journal:** Recent closed trades with search
- **Auto-refresh:** Configurable refresh interval

### Cloudflare Tunnel
When tunnel mode is active with UI mode:
1. Checks for the cloudflared binary
2. Auto-installs from Cloudflare if missing
3. Spawns cloudflared as a subprocess
4. Parses the public URL from stdout
5. Cleans up the tunnel subprocess on shutdown

**Requirements:** Outbound internet access only — no inbound ports needed beyond existing firewall rules.

### Reverse Proxy
The production nginx configuration proxies dashboard and API routes to the UI server.

### Dashboard Data Model

| Stat | Description |
|------|-------------|
| Wallet Balance | Current wallet balance |
| Available Capital | Unused capital |
| Init Balance | Starting capital |
| Total P&L | Realized plus unrealized |
| Unrealized P&L | Open position floating P&L |
| Total Positions | Cumulative entry count |
| Win Rate | Computed from win/loss count |
| Total Fees | Cumulative fee spend |

---

## 15. State Persistence

### Architecture

```
Periodic saves:
    Position monitor → periodic save → state file

On Shutdown:
    Cleanup sequence → final snapshot

On Restart:
    Check for state → load last state → restore positions → resume monitoring
```

### State Saving
The saved snapshot includes:
- Version identifier
- Available capital
- Cumulative position count
- Active positions (excluding placeholders)
- Recent closed positions
- Timestamp

Key design decisions:
- **JSONL append-only:** Crash-safe — previous line preserved if write fails
- **Placeholder filter:** Placeholder objects are excluded from persistence
- **Non-blocking I/O:** Uses asynchronous file operations

### State Loading
Static methods check for state file existence and load the last line efficiently without reading the entire file.

### State Restoration
Two-pass restoration:
1. **Pass 1:** Create Position objects for each active position, attach callbacks
2. **Pass 2:** Link hedge pairs by pair ID

Then start monitoring coroutines for each non-closed position.

**Key restoration behaviors:**
- Holding duration override uses current configuration (not stale saved values)
- All exit parameters use safe defaults for backward compatibility
- Runtime-only fields are not persisted — they start as defaults on restoration

---

## 16. Signal Handler & Shutdown Sequence

### Signal Handler Setup
Signal handlers are registered for SIGINT and SIGTERM using the asyncio event loop.

Two handler paths based on mode:

**Auto Mode** (flag or environment variable set):
- Sets running flag to false directly — immediate clean shutdown
- Bypasses pause menu entirely

**Interactive Mode:**
- Sets paused flag — triggers pause menu on next loop iteration

### Cleanup Shutdown Sequence

| Step | Action | Description |
|------|--------|-------------|
| 0a | State save | Final snapshot before connections close |
| 1 | Position close | Close all open positions |
| 2 | State save | Final snapshot after position closure |
| 3 | Final summary | Print summary statistics |
| 4 | WebSocket close | Stop all WebSocket streams |
| 5 | HTTP client close | Close API client |
| 6 | File close | Close raw and output files |
| 7 | Bytecode cleanup | Remove cache directories |

Each step has a configurable timeout. In auto-mode, positions are not closed on shutdown — they remain open on Binance and are restored on the next start via persistence.

---

## 17. Auto-Mode & Systemd Integration

### Auto-Mode Detection
The bot enters auto-mode when either the CLI flag is passed or the relevant environment variable is set.

**Effects of auto-mode:**
- Bypasses live trading confirmation prompt
- Signal handler initiates direct clean shutdown (no pause menu)
- Pause menu auto-selects close all and shutdown
- Position close is skipped in cleanup — preserves positions for next session

### Systemd Service
Auto-installed on first persistence run. Checks for existing service before installing.

**Unit file features:**
- Network-dependent startup ordering
- Simple service type
- Working directory and execution command
- Automatic restart on crash with configurable delay
- Environment variable for auto-mode detection
- Standard output/error suppression

**Key behaviors:**
- Automatic restart after any crash
- Environment variable prevents recursive systemd setup and auto-confirms
- No TTY — process survives SSH disconnect
- Starts on boot
- Recursion guard — setup exits early if environment variable is set

---

## 18. Error Handling & Recovery

### Error Classification

| Severity | Examples | Response |
|----------|----------|----------|
| 🔴 Critical | State file corruption, WebSocket disconnect | Logged, position monitoring continues via API fallback |
| 🟠 Major | Order failure | Capital returned, placeholder cleaned, logged |
| 🟡 Minor | Logger exception, stale buffer | Try/except, execution continues |
| 🔵 Info | Leverage set failure, duplicate skip | Logged, no action needed |

### Error Recovery Patterns

**1. Order failure → capital return:** If position opening fails at any point after capital deduction, capital is returned and placeholder is removed.

**2. Price buffer staleness:** Price retrieval uses a three-tier fallback: buffer (fresh) → API (live mode) → stale buffer (last resort). Positions continue monitoring with the best available data.

**3. Logger exception isolation:** All logging calls are wrapped in try/except — a logging failure never crashes a position.

**4. Stream health validation:** Last real price update tracks actual price data (not WebSocket heartbeat). Beyond a configurable staleness threshold, the system refuses to open new positions.

**5. Emergency close timeout:** If exit is triggered but the position is not closed within a configurable timeout, the monitor force-closes via direct Binance API call.

**6. Force-stop under systemd:** A pre-existing asyncio signal propagation issue under systemd may cause hangs. Use `systemctl kill -s SIGKILL` to force-stop if needed.
