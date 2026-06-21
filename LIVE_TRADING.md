# Live Trading on Binance Futures — Production Guide

**Version 3.0.0** | **June 21, 2026** | **Rebranded from Artemis**

Complete guide for running Rebirth Trader as a production trading bot with systemd persistence, auto-restart, and headless operation.

---

## Setup Guide

### First-Time Setup

```bash
# Run with --persist — auto-installs systemd service and starts trading
uv run streamer.py --scalp --all --live --persist --auto --ui --tunnel
```

On first run:
1. Creates `/etc/systemd/system/rebirth-trader.service`
2. Runs `systemctl daemon-reload && systemctl enable rebirth-trader`
3. Starts trading in the foreground with the embedded web dashboard
4. Creates a Cloudflare Tunnel for public internet access
5. On next boot or crash: systemd manages the process automatically

### Subsequent Runs

```bash
# Auto-starts via systemd. No manual command needed.
systemctl status rebirth-trader
journalctl -u rebirth-trader -f
```

### Recommended Command (Headless with Dashboard)

```bash
uv run streamer.py --scalp --all --live --persist --auto --ui --tunnel
```

The `--auto` flag ensures headless operation with no interactive prompts. The `--ui` flag starts the embedded web dashboard on port 3000, and `--tunnel` creates a Cloudflare Tunnel for public internet access.

---

## State Persistence Architecture

### Snapshot Cycle (Every 10 Seconds)

```
_periodic_save() → save_state() → write JSONL to logs/state.jsonl
```

- Append-only JSONL format — crash-safe
- Previous line preserved if write is interrupted
- All active positions serialized (31 fields each)
- Placeholder objects filtered out (only real Position objects)
- Uses `asyncio.to_thread()` for non-blocking file I/O
- **Growth rate:** ~30.5 MB/hour at 10s interval → ~5 GB/week

### State File Location
```
/home/kali/rebirth-trader/logs/state.jsonl
```

### Restoration Flow (on restart)

```python
# 1. Check if state exists
state = _load_last_state()

# 2. Two-pass restoration
# Pass 1: Create Position objects for each active position
# Pass 2: Link hedge pairs by _hedge_pair_id

# 3. Start monitoring coroutines
for position in restored_positions:
    asyncio.create_task(position.monitor())
```

**Restoration behaviors:**
- `max_holding_minutes` uses current config value (not stale saved value)
- `defensive_hedge_minutes` (1440.0 = 24h) and `candle_trail_min_ratio` (0.3) backward-compatible with saved data
- `_candle_skip_trail` runtime field defaults to `False`

---

## Systemd Management

### Unit File Location
```
/etc/systemd/system/rebirth-trader.service
```

### Commands

```bash
# Status
systemctl status rebirth-trader

# Stop (may hang — see note below)
systemctl stop rebirth-trader

# Start
systemctl start rebirth-trader

# Restart
systemctl restart rebirth-trader

# Follow logs
journalctl -u rebirth-trader -f

# Last N lines
journalctl -u rebirth-trader -n 100 --no-pager

# Logs since timestamp
journalctl -u rebirth-trader --since "1 hour ago"

# Disable auto-start
systemctl disable rebirth-trader
```

### Known: systemctl stop Hangs

Pre-existing asyncio signal propagation issue under systemd. The SIGTERM signal doesn't reach the event loop handlers. **Workaround:**

```bash
systemctl kill -s SIGKILL rebirth-trader
sleep 2
systemctl start rebirth-trader
```

This is not a regression — exists before all recent patches. SIGKILL bypasses graceful shutdown but the bot is designed to survive unclean kills via state persistence and auto-restore.

### Full Service Removal

```bash
systemctl stop rebirth-trader
systemctl disable rebirth-trader
rm /etc/systemd/system/rebirth-trader.service
systemctl daemon-reload
# Re-install on next --persist run
```

---

## Safety Features

### Multi-Layer Protection

| Layer | Protection | Details |
|-------|-----------|---------|
| Interactive confirmation | `--live` requires typed confirmation | Bypassed by `--auto` or `REBIRTH_TRADER_SYSTEMD=1` |
| API key validation | Format + permissions checked before trading | Ensures Futures API access |
| Balance capping | Live balance capped at $1000 | Prevents full account exposure |
| Capital usage cap | 90% total capital deployment | Never deploys 100% |
| Per-position cap | 2.9% max per position | Limits single-position exposure |
| Emergency close | 30s timeout then force-close | Prevents stuck positions |
| Stream health check | 60s staleness threshold | Prevents trading on dead streams |
| Trading pause window | 23:00–02:00 UTC | Blocks new entries/hedges during off-hours |
| Symbol blacklist (`--blacked`) | 3 consecutive losses → 24h auto-suspend | Prevents burning capital on failing pairs |
| State persistence | Crash-safe JSONL | Recovers positions after crash |

### Auto-Mode Behavior

When running under `--auto` or `REBIRTH_TRADER_SYSTEMD=1`:

1. **Live confirmation bypassed** — proceeds directly to trading
2. **Signal handler** sets `running = False` directly (no pause menu)
3. **Pause menu** auto-selects 'y' (close all + shutdown)
4. **Positions not closed on shutdown** — preserved for next session via state persistence

---

## Verification Checklist

Before or during live trading, verify:

- [ ] API keys have Futures trading permissions (withdraw disabled recommended)
- [ ] Account has sufficient USDT/USDC balance ($10 minimum)
- [ ] `.env` file has correct permissions (`chmod 600`)
- [ ] Systemd service is enabled and active (`systemctl is-enabled rebirth-trader`)
- [ ] `logs/` directory is writable
- [ ] State file is being written (`tail -1 logs/state.jsonl`)
- [ ] Understanding of leverage and futures risks

---

## Dashboard Monitoring

### Access URLs

| Method | URL | Notes |
|--------|-----|-------|
| Direct (LAN) | `http://<host-ip>:3000/` | Bot binds to `0.0.0.0` |
| Nginx proxy | `http://<host>/dashboard/` | Port 80, proxied to bot |
| Cloudflare Tunnel | `https://<random>.trycloudflare.com` | When `--tunnel` is active |

### Dashboard Features

- **Live P&L** hero stat with uPnL subtext
- **Stat cards:** Wallet, Realized P&L, Fees, Positions, Win Rate
- **Equity curve:** Chart.js canvas with white gradient fill
- **Active positions table:** 22+ positions with color-coded uPnL
- **Best/Worst tables:** Top and bottom performers by P&L
- **Streak dots:** Visual trade streak history (last 100 trades)
- **Trade journal:** Last 1000 closed trades with search
- **Auto-refresh:** Every 10 seconds (matches state write cycle)
- **Navigation:** Home, Dashboard, GitHub links

### API Endpoints

| Endpoint | Description |
|----------|-------------|
| `/api/stats` | Live aggregated stats (wallet, P&L, positions, fees) |
| `/api/equity` | Equity curve data points (timestamp, balance) |
| `/api/positions` | Active position details with current prices |
| `/api/history` | Closed trade history from `history.jsonl` |
| `/api/journal` | Raw full journal data |

## Monitoring Commands

```bash
# Process health
systemctl status rebirth-trader

# Live log tail
journalctl -u rebirth-trader -f

# Latest state snapshot
tail -1 logs/state.jsonl | python3 -m json.tool

# Position count
tail -1 logs/state.jsonl | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(f'Active: {sum(len(v) for v in d.get(\"active_positions\",{}).values() if isinstance(v,list))} positions')"

# Capital + P&L summary
tail -1 logs/state.jsonl | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(f'Wallet: \u0024{d.get(\"wallet_balance\",0):.2f} | P&L: \u0024{d.get(\"total_pnl\",0):.2f} | Realized: \u0024{d.get(\"realized_pnl\",0):.2f}')"

# Recent closed trades (via history.jsonl)
python3 -c "import json; lines=open('logs/history.jsonl').read().strip().split('\n'); [print(json.loads(l).get('symbol','?'),'{:.4f}'.format(json.loads(l).get('pnl_usd',0)),json.loads(l).get('exit_reason','?')) for l in lines[-10:]]"

# Trade journal summary
python3 -c "
import json
trades = [json.loads(l) for l in open('logs/history.jsonl') if l.strip()]
pnls = [t.get('pnl_usd',0) for t in trades]
wins = sum(1 for p in pnls if p > 0)
losses = sum(1 for p in pnls if p < 0)
print(f'Trades: {len(trades)} | Wins: {wins} | Losses: {losses} | WR: {wins/len(trades)*100:.1f}%' if trades else 'No trades yet')
print(f'Total P&L: \u0024{sum(pnls):.2f} | Avg Win: \u0024{sum(p for p in pnls if p>0)/wins:.2f}' if wins else '')
print(f'Fees: \u0024{sum(t.get(\"fees\",0) for t in trades):.4f}')
"

# State file growth
ls -lh logs/state.jsonl
```

---

## Log File Reference

| File | Format | Write Frequency | Contents |
|------|--------|----------------|----------|
| `logs/state.jsonl` | JSONL append | Every 10s | Active positions, capital, closed history |
| `logs/history.jsonl` | JSONL append | Per-trade-close | Permanent trade journal (symbol, pnl_usd, duration, exit reason) |
| `logs/hfscalper_YYYYMMDD_HHMMSS.jsonl` | JSONL append | Per-event (optional) | ~60 event types (open, monitor, close, errors — only with `--loghfs`) |
| `logs/raw.jsonl` | JSONL append | Per-message | Raw WebSocket data (when `--raw`) |

**Note:** `history.jsonl` populates independently of `--loghfs` — always written on position close.

---

## Known Issues

1. **Margin Type Typo (C07):** Sends `margin_type="CROSSED"` instead of `"CROSS"` — Binance silently accepts but ignores it, defaults to isolated. Not critical for trading.
2. **systemctl stop hang:** Pre-existing asyncio signal issue under systemd. Use SIGKILL.
3. **AAVEUSDC order failure (-4003):** High-priced coins may fail to meet $5 min notional with small capital. Pre-existing capital constraint — not a bot bug.
4. **CRVUSDC leverage 30 invalid:** Binance max leverage for certain symbols is below 30. Bot gracefully handles via `symbol_max_leverage` cap.
5. **Port race on restart:** Bot holds port 3000; new instance's UI fails silent if port isn't freed. Verify port free before restart.

---

## Risk Disclaimer

**Algorithmic trading involves substantial risk of loss.**

- Trading cryptocurrency futures with leverage carries high probability of total loss
- Past performance does not guarantee future results
- Start with minimum capital ($10–50) until the system is well-understood
- Monitor the system regularly — don't run unsupervised indefinitely
- This is not financial advice — educational purposes only
