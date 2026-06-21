# Features & Enhancements — v3.0.0

**Version 3.0.0** | **June 21, 2026** | **Rebranded to Rebirth Trader**

Comprehensive overview of all features, patches, and architectural improvements in the Rebirth Trader scalping bot.

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
12. [Auto-Detection Mode](#12-auto-detection-mode)
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

### What It Does
Enables crash-safe position state persistence — periodic background snapshots of all active positions are written to a local file. On restart, the system automatically restores all positions with their monitoring routines, ensuring zero interruption to active trades.

**Key Benefits:**
- Full position serialization including all tracking parameters and hedge pair references
- Crash recovery: restarting the bot seamlessly resumes monitoring of all open positions
- Append-only storage protects against corruption during unexpected shutdowns
- Efficient read mechanism avoids loading unnecessary data on restore
- Backward-compatible state loading supports older state files

---

## 2. Systemd Integration

### What It Does
Automatically installs a systemd service on first run with the persistence flag. The service ensures the bot survives SSH disconnects, crashes, and reboots — restarting automatically with an appropriate backoff.

**Key Behaviors:**
- Idempotent setup — only installs if not already registered
- Recursion guard prevents re-installation when already running under systemd
- Headless operation mode flag is appended automatically for non-interactive environments
- Runs as a simple service type with network-online dependency

---

## 3. WebSocket Migration (Binance API Change)

### What Changed
Following Binance's deprecation of legacy WebSocket endpoints in April 2026, the streaming infrastructure was migrated to the new routed endpoint architecture. Data types are now routed to their appropriate endpoints, with a fallback layer for older connections.

**Results:**
- All data streams — depth, ticker, kline, and aggregate trade — verified functioning correctly
- Graceful fallback maintains compatibility across different connection environments

---

## 4. Signal Handler & Auto-Mode Patches

### What Changed
The signal handling system was enhanced to support fully headless operation. When running in automated mode, signals trigger a clean shutdown path that bypasses interactive prompts entirely.

**Key Improvements:**
- Automated mode handles OS signals cleanly — no stuck prompts under systemd
- Final state snapshot is captured before network connections close
- Single-path shutdown sequence eliminates duplicate code paths

---

## 5. Candle Gate & Loss Recovery Trail

### Overview
A trio of interconnected candle-based features that enhance trading intelligence:

- **Loss Recovery Trail** — embedded in the stop-loss logic to salvage losing trades by exiting when the price recovers meaningfully from its worst point
- **Candle Gate** — suppresses trail activation when the candle trend favors the current position direction, letting winners run and losers recover
- **Candle Buffer** — an interval-filtered buffer that feeds both the gate and entry confirmation with clean, consistent data

**Backward Compatibility:**
All new parameters use a safe fallback pattern — older state files without these fields restore correctly without error.

---

## 6. Position-Centric Architecture

### What Changed (Major Redesign)
Each trading position now runs as an independent monitoring coroutine with fully isolated error handling. This replaced the traditional centralized loop where a single failure could cascade across all positions.

**Architecture:**
- Positions are created as independent concurrent tasks, each with its own lifecycle
- A callback pattern provides positions with controlled access to shared resources (price data, close capability, reversal detection, hedge logic)
- Positions maintain a back-reference to the engine for buffered data access

**Benefits:**
- Errors are fully isolated per position — no cascading failures
- Supports many concurrent positions, all monitoring independently
- Graceful degradation: one position can fail while others continue trading
- Independent exit logic evaluation per position

---

## 7. Six-Layer Exit Logic System

### Overview
Each position is evaluated through a priority-ordered series of exit logics. The first logic to trigger defines the exit, ensuring the most appropriate exit strategy fires first.

### Priority Order
| Priority | Logic | Description |
|----------|-------|-------------|
| 1 | **Hard Stop + Loss Trail** | Maximum-loss protection combined with a recovery-based exit that salvages partial value |
| 2 | **Take Profit** | Exits when the position reaches a target profit threshold |
| 3 | **Trailing Stop** | Locks in profits by trailing the peak price, exiting on a configurable pullback |
| 4 | **Time Limit** | Enforces a maximum holding duration to prevent capital lock-up |
| 5 | **Reversal Detection** | Uses trend analysis to exit when the market direction reverses against the position |

**Design Pattern:**
Exit logics are iterated in priority order — the first match triggers the exit. Each logic returns a structured signal containing exit price, reason, and logic identifier.

---

## 8. Price Buffer Pattern

### What Changed (Critical Performance Improvement)
Eliminated all API calls for real-time price data by caching WebSocket stream data in an in-memory buffer. Positions read prices directly from the buffer — zero exchange API calls for price retrieval.

**Performance Impact:**
- Before: The vast majority of price retrieval attempts failed due to API limitations
- After: Near-perfect price retrieval success rate
- API calls for pricing eliminated entirely — freeing rate limits for order operations

---

## 9. Exception-Safe Logging

### What Changed
All logging calls are wrapped in protective error handling. Logger exceptions no longer crash the trading system.

**Result:**
- Zero crashes from logging failures
- Trading operations continue uninterrupted even if the logging subsystem encounters errors

---

## 10. Placeholder & Ghost Position Prevention

### Overview
A multi-layered system that prevents ghost positions — trades recorded in state but not actually open on the exchange.

### Prevention Mechanisms
1. **Slot reservation:** A lightweight placeholder reserves the position slot *before* the order is placed, replaced by a real position only after the order fills
2. **Capital validation:** Capital is verified before placeholder creation — an order failure automatically returns capital
3. **State persistence filter:** Placeholders are naturally excluded from state snapshots
4. **Error path cleanup:** All exception handlers clean up placeholders and return capital to the available pool

---

## 11. Emergency Shutdown & Cleanup

### Cleanup Sequence
An orderly multi-step shutdown process ensures all resources are released cleanly:

| Step | Action | Description |
|------|--------|-------------|
| 1 | State save | Final snapshot before connections close |
| 2 | Position closure | Close all positions via exchange API |
| 3 | State save | Final state after position closure |
| 4 | Final summary | Print trade summary report |
| 5 | WebSocket close | Graceful stream disconnection |
| 6 | HTTP close | Exchange client disconnect |
| 7 | File handles | Close all output files |
| 8 | Bytecode cleanup | Remove cached bytecode |

### Auto-Mode Behavior
When running in automated headless mode:
- Position closure is skipped — positions remain open on the exchange
- State is saved so the next startup restores monitoring
- Enables crash recovery without unnecessary position closure

---

## 12. Auto-Detection Mode

### What Changed
Removed the separate demo flag. The system now auto-detects trading mode:

| Command | Mode | Behavior |
|---------|------|----------|
| Console mode | Simulated | Paper trading with no real exchange orders |
| Live mode | Live | Real orders on the exchange |

Detection is based on whether a live exchange client connection is available — no manual flag required.

---

## 13. Concurrent Position Management

### Overview
A configurable limit on the number of concurrent non-hedge positions. Hedge positions bypass this limit, ensuring protection trades are never blocked.

**Enforcement:**
- Non-hedge, non-placeholder positions are counted against the limit
- Hedge positions are allowed even when the limit is reached
- New entry signals are cleanly rejected when the limit is hit

---

## 14. Stream Health Validation

### What It Does
Tracks when actual price data (not connection keepalives) was last received per trading symbol. Before opening a new position, validates that the data stream is healthy.

**Fix:**
Prevents opening positions on stalled or dead streams, eliminating the risk of trading on stale data.

---

## 15. Candle Buffer — Interval-Filtered & Cleaned

### What Changed
The candle buffer was redesigned to only accept closed candles from a single, configured kline interval. Previously, candles from all subscribed intervals were mixed into the same buffer, making trend calculations mathematically invalid.

**Improvements:**
- Clean, single-interval data ensures mathematically valid trend calculations
- Configurable buffer size and trend parameters
- Cold-start seeding: On startup, historical candle data is fetched to fill the buffer immediately — eliminating the long warm-up period previously required
- Seeding completes quickly even for large symbol sets using controlled concurrent requests
- Seamless transition from seeded data to live WebSocket candle closes

**Integration:**
Seeding runs before state restoration, so recovered positions immediately have full candle context with no data gap.

---

## 16. Loss Recovery Trail

### Concept
The loss-side counterpart to the trailing profit stop. While the profit trail locks in gains, the loss trail salvages value from losing trades by exiting when the price recovers a meaningful distance from its worst point.

### How It Works
- Tracks the deepest loss (trough) reached during a position
- When the price recovers a configurable distance from the trough, triggers an exit
- The recovery exit preserves capital that would otherwise be lost to a full hard stop
- Works in conjunction with the candle gate to avoid exiting during temporary pullbacks in a favorable trend

---

## 17. Candle Gate — Profit & Loss Trail Suppression

### What It Does
In every monitoring cycle, compares the candle trend against the position direction. When the trend supports the position, trail activation is suppressed — letting profitable positions run further and giving losing positions room to recover.

### Key Design Decisions
- The gate evaluates fresh each cycle based on current candle data
- Not persisted to state — trails re-evaluate after restart
- Hedge positions always bypass the gate
- The hard stop always fires regardless of gate state

---

## 18. Defensive Hedge with Candle Gate

### What It Does
After a configurable period of consecutive losses, evaluates whether to open a defensive hedge position. Before hedging, verifies that the candle trend actually opposes the current position — preventing unnecessary hedging during favorable market conditions.

---

## 19. Capital Capping & Balance Protection

### Live Balance Capping
The system applies a configurable upper limit on deployed capital, protecting against excessive exposure even on larger accounts.

### Capital Allocation
A hierarchical allocation system ensures responsible capital deployment:
- Balance is subject to a maximum cap
- A deployment ratio limits total capital at risk
- Per-position sizing spreads risk across multiple trades

---

## 20. Order Failure Recovery & Auto-Scale

### Auto-Scale on Rejection
When an exchange rejects a market order due to insufficient notional value, the system automatically calculates the minimum viable order size and retries once — maximizing capital utilization without manual intervention.

### Capital Return on Failure
All order failure paths automatically return allocated capital to the available pool, ensuring no capital is permanently locked by failed order attempts.

---

## 21. Files Changed Reference

### Source Files

| File | Primary Role |
|------|-------------|
| `streamer.py` | CLI entry point, systemd setup, argument parsing |
| `src/rebirth_trader/cli.py` | CLI orchestration, signal handlers, cleanup, state restore, tunnel startup |
| `src/rebirth_trader/hfscalper.py` | Core trading engine: positions, exits, state, buffers |
| `src/rebirth_trader/binance_client.py` | Exchange API wrapper |
| `src/rebirth_trader/config.py` | Configuration loader |
| `src/rebirth_trader/__init__.py` | Version and package metadata |
| `src/rebirth_trader/ui/server.py` | Embedded web dashboard HTTP server |
| `src/rebirth_trader/ui/index.html` | Single-file dashboard HTML+CSS+JS template |

### Documentation Files

| File | Purpose |
|------|---------|
| `docs/README.md` | Quick-start, CLI reference, dashboard URLs, key features |
| `docs/ARCHITECTURE.md` | Full system architecture including dashboard and tunnel |
| `docs/LIVE_TRADING.md` | Production trading setup, dashboard monitoring, API reference |
| `docs/FEATURES_ENHANCEMENT.md` | This file — all features and patches |

### Runtime Files

| File | Purpose |
|------|---------|
| `logs/state.jsonl` | State persistence (periodic position snapshots) |
| `logs/history.jsonl` | Permanent trade journal (one entry per close) |
| `logs/hfscalper_*.jsonl` | Trading event logs (optional) |
| `logs/raw.jsonl` | Raw WebSocket data (optional) |
| `.env` | API credentials (restricted permissions) |

### System Files

| File | Purpose |
|------|---------|
| Systemd unit | Service definition with automated restart |
| Nginx config | Reverse proxy for dashboard and API endpoints |
| Landing page | Static HTML entry point linking to dashboard |
| Cloudflare Tunnel binary | Auto-installed tunnel client |
| Python virtual environment | Managed Python dependency environment |

---

## 22. Trading Pause Window

### What It Does
Creates a configurable daily window during which new trade entries and defensive hedges are blocked. Existing positions continue normal monitoring — all exit logics remain active.

### Behavior

| Scenario | During Pause | Outside Pause |
|----------|-------------|---------------|
| New entry signal | Blocked | Normal evaluation |
| Defensive hedge | Blocked | Normal evaluation |
| Active position monitoring | Continues normally | Continues normally |
| Stop loss / trailing stop | Active | Active |
| Position close on exit | Active | Active |

### Rationale
Reduces risk during periods of lower liquidity and higher volatility. The pause window can be configured to align with typical low-volume market hours.

---

## 23. Symbol Blacklist (`--blacked`)

### What It Does
Auto-suspends trading on symbols that hit a configurable number of consecutive losses, preventing the bot from repeatedly losing capital on underperforming pairs. Supports custom cooldown durations and pre-blacklisting of known-problematic symbols.

### CLI Usage
- Enable blacklist with default cooldown
- Enable with custom cooldown duration
- Pre-blacklist specific symbols with custom cooldown

### Loss Tracking & Auto-Suspend
After each position close with a loss, the system tracks consecutive losses per symbol. When the threshold is reached, the symbol is automatically blacklisted for the configured cooldown period.

### Persistence
Blacklist state is saved and restored across restarts as part of the state persistence system.

### Cooldown Expiry
Expired blacklist entries are automatically cleaned up during market assessment cycles.

### Relationship to Other Features
- Trading pause is checked first, then blacklist — both must pass for a new entry
- State persistence saves and restores blacklist state across restarts
- Defensive hedge is gated by blacklist — won't hedge a blacklisted symbol

---

## 24. Dashboard UI (`--ui`)

### What It Does
Starts an embedded web dashboard server when the `--ui` flag is passed. The dashboard shows real-time bot statistics, active positions, equity curve, and trade journal — accessible from any device on the local network.

### Dashboard Components
- **Stat cards** — real-time metrics including P&L, active positions, win rate, wallet balance, realized P&L, and fees
- **Equity curve** — interactive chart with auto-refresh
- **Positions table** — active positions with color-coded unrealized P&L
- **Best/Worst tables** — top and bottom performers by P&L
- **Streak visualization** — visual streak tracker with hover details
- **Trade journal** — sortable, searchable table of recent trades

### API Endpoints
The dashboard is powered by a lightweight REST API:

| Endpoint | Description |
|----------|-------------|
| `/api/stats` | Aggregated statistics |
| `/api/equity` | Equity curve data |
| `/api/positions` | Active positions with current prices |
| `/api/history` | Closed trade history |
| `/api/journal` | Full trade journal |

### Nginx Integration
The production nginx configuration proxies the dashboard and API through standard web ports, serving the dashboard alongside any existing web infrastructure.

### History
- Redesigned with modern glass-morphism aesthetic
- Consolidated from separate dashboard servers to a single embedded server
- Landing page integrated with live API data instead of stale files

### Notes
- The dashboard server binds to all network interfaces for LAN access
- Runs in a background thread without blocking trading operations
- The HTML template is loaded at startup

---

## 25. Cloudflare Tunnel (`--tunnel`)

### What It Does
When `--tunnel` is passed alongside `--ui`, the bot creates a secure Cloudflare Tunnel for public internet access to the dashboard. The tunnel is spawned as a managed subprocess — automatically downloading the tunnel client if missing, and cleaning up cleanly on shutdown.

### How It Works
- The tunnel establishes an outbound connection to Cloudflare's edge network
- A public URL is generated and displayed for dashboard access
- No inbound port forwarding is required — the connection is outbound only
- The tunnel process is automatically terminated on bot shutdown

### Security
- **Outbound only:** No inbound port forwarding required
- **Firewall compatible:** Works with restrictive firewall configurations blocking all but essential ports
- **Ephemeral URLs:** Each restart generates a new temporary URL (permanent URL available with domain setup)

### URLs

| Scope | Description |
|-------|-------------|
| Temporary (free) | Auto-generated URL, changes on each restart |
| Permanent | Custom subdomain via DNS configuration |

### Notes
- If the bot crashes hard, the tunnel subprocess may orphan — manual cleanup may be needed
- Each restart generates a new temporary URL unless a permanent tunnel is configured

---

**Version:** 3.0.0 | **Last Updated:** June 21, 2026 | **Status:** Rebranded to Rebirth Trader
