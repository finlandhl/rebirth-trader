# Live Trading — Production Guide

> Developed and refined with real capital across live market conditions. Every strategy, exit logic, and safeguard was battle-tested against actual Binance Futures order flow — not historical simulations.

## Deployment

Deploying Rebirth Trader for live trading is a one-command process that:

- Installs the bot as a system-managed service (on servers) or runs as a background process (on mobile via Termux/iSH)
- Enables automatic startup on boot and restart on failure
- Starts the embedded web dashboard for real-time monitoring
- Optionally provisions a secure tunnel for remote access
- Operates fully headless once configured

After initial setup, the bot manages itself. No manual commands needed for normal operation.

## State Persistence

The bot continuously saves its operational state to disk using a crash-safe append-only format:

- All active positions are serialized with full metadata
- State writes are non-blocking — trading performance is unaffected
- On restart, the bot auto-detects saved state and restores all positions
- Hedge relationships are re-established in a two-pass restoration
- Monitoring resumes automatically for all recovered positions

## Service Management

Standard service operations are supported:

- **Status** — Verify the service is running and healthy
- **Stop** — Gracefully stop trading
- **Start** — Start or resume the service
- **Restart** — Full restart with clean state
- **Logs** — Stream or inspect recent log entries

## Dashboard Monitoring

The embedded web UI provides real-time access to:

- **Live aggregated stats** — Wallet balance, realized P&L, win rate, active positions
- **Equity curve** — Historical balance chart with interactive display
- **Active positions** — Entry price, current P&L, duration, direction
- **Trade history** — Closed trades with exit reasons
- **Internationalization** — Configurable language support

## Remote Access

With the tunnel option enabled, the dashboard becomes accessible from anywhere via a secure Cloudflare Tunnel. No port forwarding or static IP required.

## Data Management

Trade data is stored in two append-only files:

- **State file** — Periodic snapshots of active positions (for crash recovery)
- **History file** — Permanent record of every closed trade with PnL and exit reason

Both formats are human-readable and designed to prevent data loss on unexpected shutdowns.

## Prerequisites

- A Binance Futures account
- API key with futures trading permissions
- Any device with a Linux environment or CLI terminal — VPS, server, laptop, or mobile phone (tested on iOS via iSH/Termius and Android via Termux)
- Internet connectivity

## Risk Management

- All trading is subject to configurable capital protection limits
- Live mode is explicitly enabled — simulation mode by default prevents accidental real trading
- Systemd service ensures automatic recovery on crashes
- State persistence prevents position loss on restart

---

⚠ **Risk Warning:** Trading cryptocurrencies carries significant financial risk. Past performance does not guarantee future results. Use at your own risk.
