# Live Trading — Production Guide

> Developed and refined with real capital across live market conditions. Every strategy, exit logic, and safeguard was battle-tested against actual Binance Futures order flow — not historical simulations.

## Deployment

Deploying Rebirth Trader for live trading is a one-command process that:

- Installs the bot as a system-managed service (on servers) or runs as a background process (on mobile via Termux/iSH)
- Enables automatic startup on boot and restart on failure
- Starts the embedded web dashboard for real-time monitoring
- Optionally provisions a secure tunnel for remote access
- Operates fully headless once configured

```bash
uv run streamer.py --scalp --all --live --persist --auto --ui --tunnel
```

After initial setup, the bot manages itself. No manual commands needed for normal operation.

## State Persistence

The bot continuously saves its operational state to disk every **20 seconds** using a crash-safe append-only format:

- All active positions are serialized with full metadata — state writes are non-blocking
- On restart, the bot auto-detects saved state and restores all positions
- Hedge relationships are re-established in a two-pass restoration
- Monitoring resumes automatically for all recovered positions
- Leaderboard ranking data is saved and restored

## Service Management

```bash
systemctl status rebirth-trader          # Health check
journalctl -u rebirth-trader -f          # Follow live logs
journalctl -u rebirth-trader -n 100      # Last 100 lines
systemctl stop rebirth-trader            # Stop
systemctl start rebirth-trader           # Start
systemctl restart rebirth-trader         # Full restart
systemctl disable rebirth-trader         # Remove auto-start
systemctl kill -s SIGKILL rebirth-trader # Force kill (if stop hangs)
```

## Dashboard Monitoring

The embedded web UI provides real-time access to:

- **Live aggregated stats** — Wallet balance, realized P&L, win rate, active positions, best/worst trade
- **Equity curve** — Historical balance chart with interactive display (incremental reads)
- **Active positions** — Entry price, current P&L, duration, direction, lifecycle bar
- **Lifecycle bar** — Position age, profit/loss/trail status at a glance
- **Trade history** — Closed trades with exit reasons from history.jsonl
- **Streak dots** — Visual trade streak history (last 100 trades)
- **Auto-refresh** every 10s matching state write cycle

## Remote Access

With the tunnel option enabled, the dashboard becomes accessible from anywhere via a secure Cloudflare Tunnel. No port forwarding or static IP required.

```bash
# The bot auto-spawns cloudflared when --tunnel is used:
# cloudflared tunnel --url http://localhost:8080
# Auto-installs if missing. Clean shutdown on exit.
```

## Data Management

Trade data is stored in two append-only files:

- **State file** (`logs/state.jsonl`) — Periodic snapshots every 20s of active positions (for crash recovery). Includes capital, realized PnL, blacklist, leaderboard ranking.
- **History file** (`logs/history.jsonl`) — Permanent record of every closed trade with realized PnL (dollar value), fees, exit reason, direction, and leverage. Used for leaderboard ranking recovery on restart.

Both formats are human-readable and designed to prevent data loss on unexpected shutdowns.

## Prerequisites

- A Binance Futures account
- API key with futures trading permissions
- Any device with a Linux environment or CLI terminal — VPS, server, laptop, or mobile phone (tested on iOS via iSH/Termius and Android via Termux)
- Internet connectivity
- **Minimum VPS spec:** 1 vCPU, 8GB RAM, 10GB storage (+50GB recommended for debug logging)

## Risk Management

- All trading is subject to configurable capital protection limits (risk_per_trade=2.7%, max_total_capital_usage=90%)
- Live mode is explicitly enabled via `--live` — simulation mode by default prevents accidental real trading
- Systemd service ensures automatic recovery on crashes
- State persistence prevents position loss on restart
- **LimitNOFILE=65536** applied via systemd override for 500+ symbol streaming
- Trading pause window (23:00–02:00 UTC) blocks new entries in low-liquidity hours

---

⚠ **Risk Warning:** Trading cryptocurrencies carries significant financial risk. Past performance does not guarantee future results. Use at your own risk.
