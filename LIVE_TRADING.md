# Live Trading on Binance Futures — Production Guide

**Version 3.0.0** | **June 21, 2026**

Production deployment guide for running Rebirth Trader as a headless, persistent trading service with automated recovery and process supervision.

---

## Setup Overview

### First-Time Deployment

The bot can be deployed with a single command that handles the full setup pipeline:

- **Service installation** — Automatically creates and registers a system service unit
- **Process supervision** — Enables auto-start on boot and restart on failure
- **Trading dashboard** — Starts an embedded web interface for real-time monitoring
- **Remote access** — Optionally provisions a secure tunnel for public internet access
- **Headless mode** — Operates without interactive prompts once configured

On initial deployment, the setup procedure:
1. Registers the bot as a system-managed service
2. Reloads the service manager and enables auto-start
3. Begins trading with the embedded monitoring dashboard
4. After initial configuration, the bot automatically resumes on system boot or process restart

### Subsequent Operation

After initial deployment, the bot manages itself. No manual commands are needed for normal operation. Service status and logs can be inspected through standard service management tools.

---

## State Persistence Architecture

### Periodic State Capture

The bot captures its operational state on a regular cadence:

- Append-only format — crash-safe by design
- Previous state preserved if a write operation is interrupted
- All active positions serialized with full metadata
- Transient placeholder objects are excluded (only active positions are persisted)
- Non-blocking file I/O ensures trading performance is unaffected

### State File

State is written to a dedicated file within the application's log directory. The format supports incremental growth and is human-readable for inspection purposes.

### Restoration Flow (on restart)

When the bot restarts, it automatically recovers its previous session:

1. **State detection** — Checks for a saved state from the previous session
2. **Two-pass restoration** — First restores individual positions, then re-establishes hedge relationships
3. **Monitoring resumption** — Restarts position monitoring coroutines for all recovered positions

**Recovery behaviors:**
- Time-based parameters (e.g., max holding duration) use current configuration values, not stale saved values
- Runtime flags default to standard values for backward compatibility
- Configurable thresholds are restored from the active configuration

---

## Service Management

### Service Unit Location

The service unit is registered in the system's standard service directory.

### Common Operations

| Operation | Description |
|-----------|-------------|
| Status check | Verify the service is running and healthy |
| Stop | Gracefully stop the service (may require force-clean in some environments) |
| Start | Start or resume the service |
| Restart | Stop and start the service |
| Log streaming | Follow live log output |
| Log review | Inspect recent log entries |
| Timestamp-filtered logs | Review logs from a specific time range |
| Disable auto-start | Prevent the service from starting on boot |

### Known: Stop May Require Force-Clean

In certain environments, the standard stop signal may not propagate correctly to asynchronous event handlers. The recommended workaround is:

1. Send a force-termination signal to the service
2. Allow a brief cooldown period
3. Restart the service normally

This behavior is inherent to certain process supervision environments and is not a regression. The bot is designed to survive unclean shutdowns through its state persistence and automatic restoration capabilities.

### Full Service Removal

To completely unregister the service:
1. Stop the service
2. Disable auto-start
3. Remove the service unit file
4. Reload the service manager

The service will be re-registered automatically on the next deployment run.

---

## Safety Features

### Multi-Layer Protection

The bot implements a defense-in-depth approach to risk management:

| Layer | Protection | Details |
|-------|-----------|---------|
| Interactive confirmation | Live trading requires explicit typed confirmation | Can be bypassed in headless/auto mode |
| API key validation | Format and permissions validated before trading | Ensures correct exchange access |
| Balance capping | Live trading capital is capped at a configurable maximum | Prevents full account exposure |
| Capital usage limit | A configurable percentage cap on total capital deployment | Never deploys full balance |
| Per-position cap | Individual position size limited to a configurable maximum | Limits single-exposure risk |
| Emergency close | Force-close timeout for stuck positions | Prevents indefinite hanging |
| Stream health monitoring | Staleness detection for market data streams | Prevents trading on stale data |
| Trading windows | Scheduled pause periods blocking new entries | Configurable off-hours blocking |
| Symbol blacklist | Auto-suspension after consecutive losses | Prevents burning capital on failing pairs |
| State persistence | Crash-safe state file format | Recovers positions after unexpected shutdown |

### Auto-Mode Behavior

When running in headless or auto mode:

1. **Live confirmation bypassed** — Proceeds directly to trading without interactive prompts
2. **Signal handling** — Shutdown signal triggers immediate safe state transition (no pause menu)
3. **Pause menu auto-selection** — Auto-selects safe shutdown sequence
4. **Position preservation** — Active positions are not closed on shutdown; they persist for the next session via state persistence

---

## Pre-Flight Verification Checklist

Before or during live trading, verify:

- [ ] API keys have the correct exchange permissions (withdraw disabled recommended)
- [ ] Account has sufficient balance above the configured minimum threshold
- [ ] Environment configuration file has correct file permissions
- [ ] Service is registered, enabled, and active
- [ ] Log directory is writable
- [ ] State file is being written and populated
- [ ] Understanding of leverage and futures trading risks

---

## Dashboard Monitoring

### Access Methods

| Method | Notes |
|--------|-------|
| Direct (LAN) | Local network access via the bot's host IP |
| Reverse proxy | Access through a web server proxy for unified URL namespace |
| Secure tunnel | Public internet access via auto-provisioned tunnel |

### Dashboard Features

- **Live P&L** — Real-time profit and loss display with unrealized P&L subtext
- **Stat cards** — Wallet balance, realized P&L, fees, active positions, win rate
- **Equity curve** — Interactive chart showing balance over time
- **Active positions table** — Current positions with color-coded performance indicators
- **Best/Worst performers** — Top and bottom positions by P&L
- **Trade streak history** — Visual representation of recent trade outcomes
- **Trade journal** — Searchable history of closed trades
- **Auto-refresh** — Dashboard updates on a regular cadence matching the state write cycle

### API Endpoints

| Endpoint | Description |
|----------|-------------|
| `/api/stats` | Live aggregated statistics (wallet, P&L, positions, fees) |
| `/api/equity` | Equity curve data points over time |
| `/api/positions` | Active position details with current prices |
| `/api/history` | Closed trade history |
| `/api/journal` | Full raw journal data |

---

## Monitoring Commands

Standard system tools can be used to inspect the bot's health:

- **Process health** — Check service status for uptime and running state
- **Live log tail** — Stream real-time log output
- **Latest state snapshot** — Inspect the most recent persisted state
- **Position count** — Query active position count from the state file
- **Capital and P&L summary** — Extract wallet balance, total P&L, and realized P&L
- **Recent closed trades** — Query recent trade history
- **Trade journal summary** — Aggregate statistics from the trade history (total trades, win rate, average win/loss, fees)
- **State file growth** — Monitor the size of the persistent state file

---

## Log File Reference

| File | Format | Write Frequency | Contents |
|------|--------|----------------|----------|
| State file | Append-only structured log | Periodic (configurable) | Active positions, capital, closed history |
| Trade history | Append-only structured log | Per trade close | Permanent trade journal (symbol, P&L, duration, exit reason) |
| Event log | Append-only structured log | Per event (optional) | Detailed event types for debugging (open, monitor, close, errors) |
| Raw data log | Append-only structured log | Per message (optional) | Raw exchange data for analysis |

**Note:** The trade history file is populated independently of optional debug logging — always written on position close.

---

## Known Issues

1. **Service stop signal propagation** — In certain environments, the stop signal may not reach event loop handlers properly. Use force-termination as a workaround.
2. **High-value asset order limits** — High-priced coins may fail minimum notional requirements with small capital. This is a capital constraint, not a bot issue.
3. **Leverage limits per symbol** — Exchange-imposed maximum leverage varies by symbol. The bot gracefully handles this via automatic leverage capping.
4. **Port binding on restart** — If the service is restarted before the port is released, the dashboard may fail to start. Verify port availability before restarting.

---

## Risk Disclaimer

**Algorithmic trading involves substantial risk of loss.**

- Trading cryptocurrency futures with leverage carries high probability of total loss
- Past performance does not guarantee future results
- Start with minimum capital until the system is well-understood
- Monitor the system regularly — do not run unsupervised indefinitely
- This is not financial advice — educational purposes only
