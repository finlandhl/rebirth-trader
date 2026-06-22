# Rebirth Trader — Binance Futures Scalping System

**Version 3.0.0** | **Production Ready** | **Systemd-Managed**

A high-frequency Binance Futures scalping bot with state persistence, systemd auto-restart, embedded live dashboard, hedge-mode position management, and multi-layer exit logic.

- **Repository:** `/home/kali/rebirth-trader/`
- **Runtime:** Python 3.13, uv package manager, systemd service
- **Status:** ✅ Actively trading
- **Website:** https://rebirthtrader.com
- **Dashboard:** https://rebirthtrader.com/dashboard/
- **Public Tunnel:** Cloudflare Tunnel via `--tunnel` flag

---

## Quick Start

### Install Dependencies
```bash
cd /home/kali/rebirth-trader
uv sync
```

### Set Up API Keys
```bash
cat > .env << 'EOF'
BINANCE_API_KEY=your_api_key
BINANCE_API_SECRET=your_api_secret
EOF
chmod 600 .env
```

### Console Mode (No Real Trading)
```bash
uv run streamer.py --scalp --symbols BTCUSDC
```

### Live Trading (Full Persistence + Dashboard)
```bash
# First run — installs systemd service, starts trading with dashboard
uv run streamer.py --scalp --all --live --persist --auto --ui --tunnel
```

### Subsequent Runs (after reboot)
```bash
# Auto-starts via systemd. No manual command needed.
systemctl status rebirth-trader
journalctl -u rebirth-trader -f
```

---

## CLI Flags Reference

| Flag | Type | Description |
|------|------|-------------|
| `--scalp` | bool | Enable HFScalper trading mode |
| `--live` | bool | Real money trading on Binance Futures |
| `--persist` | bool | Save/restore state + auto-install systemd service |
| `--auto` / `-a` | bool | Bypass all confirmations |
| `--ui` | bool | Start embedded web dashboard on port 3000 |
| `--tunnel` | bool | Create a Cloudflare Tunnel for public internet access (requires `--ui`) |
| `--loghfs` | bool | Enable JSONL event logging (legacy — may need restart cycle) |
| `--symbols` | str | Comma-separated symbols (mutually exclusive with `--all`) |
| `--all` | bool | All tradable futures symbols (mutually exclusive with `--symbols`) |
| `--save` | str | Save stream to a file |
| `--raw` | bool | Save raw WebSocket JSON |
| `--blacked` | str[*] | Symbol blacklist with configurable cooldown |
| `--version` | - | Show version number and exit |
| `-h, --help` | - | Show this help message and exit |

---

## Architecture Overview

```
streamer.py → RebirthTraderCLI → HFScalper → Position.monitor() (per-position coroutine)
                                          ↓
                              UI Server (port 3000)
                                ↓ nginx proxy
                          /dashboard/ + /api/ → rebirthtrader.com
                                ↓ cloudflared
                          TryCloudflare URL (--tunnel)
```

**Core Design: Position-Centric Architecture**
- Each position runs as an independent `asyncio` coroutine
- Errors are isolated per position — no cascade failures
- Shared resources (price buffer, logger) accessed via callbacks
- 20 concurrent positions maximum (hedge positions excluded from limit)

**Data Flow:**
```
Binance WebSocket streams
    → process_stream_data()
    → price_buffer[symbol] (deque, maxlen=120)
    → candle_buffer[symbol] (deque, maxlen=100, 5m interval, REST-seeded at startup)
    → assess_market_conditions() (EMA 50/100 trend + RSI + volatility)
    → Kline confirmation (EMA 12/26 at 5m candle_kline_interval)
    → calculate_position() (sizing, capital)
    → open_position() (placeholder → order → position)
    → Position.monitor() (exit logic loop + candle gate check)
```

**Data Sources:**
- **Price buffer:** ticker + trade + kline WebSocket data → zero API calls
- **Candle buffer:** 5m closed candles only (filtered by `candle_kline_interval`, REST-seeded at startup) → trend assessment
- Both buffers are in-memory deques — no disk I/O during trading

---

## Risk Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `risk_per_trade` | 2.7% | Capital risk per position |
| `max_capital_per_position` | 2.9% | Max capital deployed per position |
| `max_total_capital_usage` | 90% | Total capital deployment ceiling |
| `hard_stop_loss` | 4% | Absolute stop loss from entry |
| `loss_trail_activation` | 3.5% | Drawdown threshold for loss recovery trail |
| `loss_trail_distance` | 1.1% | Recovery from trough triggering loss exit |
| `profit_target` | 6% | Take-profit target |
| `trail_activation` | +2.5% | Profit threshold activating trailing stop |
| `trail_distance` | 0.092% | Trailing stop distance (pullback from peak) |
| `trail_min_ratio` | 0.3 | Min PnL ratio of trail activation before candle gate triggers |
| `max_holding_minutes` | 10080 (7 days) | Time limit per position |
| `max_concurrent_positions` | 20 | Position count ceiling (hedges excluded) |
| `max_leverage` | 30x | High confidence (confidence > 0.8) |
| `default_leverage` | 20x | Medium confidence (0.5 < conf ≤ 0.8) |
| `min_leverage` | 10x | Low confidence (conf ≤ 0.5) |

---

## Exit Logic — Priority Order

| Priority | Logic | Trigger |
|----------|-------|---------|
| 1 | **Stop Loss** (hard) | Price crosses 4% stop |
| — | **Loss Recovery Trail** | 3.5% trough + 1.1% recovery (checks before hard stop) |
| 2 | **Take Profit** | +6% target hit |
| 3 | **Trailing Stop** | 1.3% peak then 0.092% pullback |
| 4 | **Time Limit** | 7 days max hold |
| 5 | **Reversal** | Trend reversal with confidence |

Loss trail and trailing stop are both **candle-gated** — suppressed when 5m EMA trend matches position direction.

---

## Dashboard & Monitoring

### Embedded Web Dashboard (`--ui`)
The bot serves a real-time monitoring dashboard from its built-in UI server:
- **Direct:** `http://<host>:3000/` (binds to `0.0.0.0` for LAN access)
- **Nginx Proxy:** https://rebirthtrader.com/dashboard/
- **Tunnel:** `https://<random>.trycloudflare.com` (when `--tunnel` is used)

The dashboard displays:
- **Live P&L** (hero stat), Wallet, Realized P&L, Fees
- **22+ active positions** with real-time uPnL
- **Equity curve** (white gradient fill, Chart.js)
- **Trade journal** — last 1000 closed trades
- **Session summary** — win rate, positions, avg duration
- **Streak dots** — visual trade streak history (last 100)
- **Auto-refresh** every 10s matching state write cycle

### API Endpoints (proxied via nginx `/api/`)
| Endpoint | Response |
|----------|----------|
| `GET /api/stats` | Live aggregated stats (capital, P&L, positions, fees) |
| `GET /api/equity` | Equity curve data points (timestamp, balance) |
| `GET /api/positions` | Active position details |
| `GET /api/history` | Closed trade history from `history.jsonl` |
| `GET /api/journal` | Raw full journal data |

### Cloudflare Tunnel (`--tunnel`)
When `--tunnel` is passed alongside `--ui`, the bot:
1. Detects cloudflared at `/usr/local/bin/cloudflared`
2. Auto-installs it if missing (via `_install_cloudflared()`)
3. Spawns `cloudflared tunnel --url http://localhost:3000` as a subprocess
4. Parses and prints the public TryCloudflare URL on startup
5. Kills the tunnel subprocess cleanly on shutdown
6. Requires only outbound connectivity (UFW inbound rules unaffected)

**Custom domain:** https://rebirthtrader.com — public website with Cloudflare SSL.

---

## Systemd Management

```bash
systemctl status rebirth-trader          # Health check
journalctl -u rebirth-trader -f          # Follow live logs
journalctl -u rebirth-trader -n 100      # Last 100 lines
journalctl -u rebirth-trader --since "1 hour ago"  # Recent output
systemctl stop rebirth-trader            # Stop (use SIGKILL if hanging)
systemctl start rebirth-trader           # Start
systemctl restart rebirth-trader         # Full restart
systemctl disable rebirth-trader         # Remove auto-start
```

**Known:** `systemctl stop` may hang (SIGTERM → asyncio signal propagation issue). Use `systemctl kill -s SIGKILL` as workaround.

---

## Monitoring Commands

```bash
# Latest state snapshot
tail -1 logs/state.jsonl | python3 -m json.tool

# Position count
tail -1 logs/state.jsonl | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(f'Active: {sum(len(v) for v in d.get(\"active_positions\",{}).values() if isinstance(v,list))} positions')"

# Capital + P&L summary
tail -1 logs/state.jsonl | python3 -c "import sys,json; d=json.loads(sys.stdin.read()); print(f'Capital: {d.get(\"capital\",\"?\")} | Realized: {d.get(\"realized_pnl\",\"?\")}')"

# Trade journal tail
python3 -c "import json; lines=open('logs/history.jsonl').read().strip().split('\n'); [print(json.loads(l).get('symbol','?'),json.loads(l).get('pnl_usd',0)) for l in lines[-5:]]"

# State file size & growth
ls -lh logs/state.jsonl
```

---

## Key Features

- **State Persistence (`--persist`):** JSONL crash-safe snapshots every **10s** to `logs/state.jsonl`. Full position restoration on restart with hedge pair linking.
- **Systemd Integration:** Auto-installs `rebirth-trader.service` with `Restart=always`, survives SSH disconnect, reboot, and crashes.
- **Embedded Dashboard (`--ui`):** Real-time web dashboard on port 3000 with live P&L, equity curve, positions table, trade journal, and auto-refresh (10s).
- **Cloudflare Tunnel (`--tunnel`):** Public internet access via TryCloudflare — auto-installs cloudflared, subprocess lifecycle managed by the bot.
- **Nginx Integration:** `/dashboard/` and `/api/` proxied through port 80 nginx server for the main landing page.
- **Hedge Mode:** Dual-side positions (long + short per symbol) enabled via Binance Hedge Mode, with defensive hedge gate (candle-confirmed).
- **6-Layer Exit System:** Stop loss → take profit → trailing stop → time limit → reversal detection + loss recovery trail, checked in strict priority order.
- **Loss Recovery Trail:** Embedded in StopLossExit — salvages losing trades by exiting when price recovers 1.1% from a 3.5%+ trough.
- **Candle Gate (Profit & Loss):** Suppresses trailing stop activation when 5m candle trend matches position direction. Works for both profit trails and loss recovery trails.
- **Candle Buffer (Interval-Filtered):** Clean uniform 5m data for trend assessment — only `candle_kline_interval` closed candles populate the buffer.
- **Price Buffer Pattern:** Zero API calls for price data — all positions read from in-memory WebSocket buffer (maxlen=120).
- **Multi-Factor Reversal Detection:** Pattern (local extrema) + momentum (3 consecutive closes) + volume confirmation.
- **Auto-Mode:** `--auto` flag or `REBIRTH_TRADER_SYSTEMD=1` bypasses interactive prompts for headless operation.
- **Trading Pause Window:** Halts new entries and hedges during 23:00–02:00 UTC. Existing positions continue normal monitoring.
- **Symbol Blacklist (`--blacked`):** Auto-suspends symbols after 3 consecutive losses for 24h (configurable via NH suffix). Pre-blacklist symbols on startup. Persisted across restarts via state.jsonl. Gates both new entries and defensive hedges.
- **Capital Capping:** Live balance capped at $1000 even on larger accounts.
- **Stream Health Validation:** `last_real_price_update` tracking prevents opening positions on dead WebSocket streams.
- **Interval-Filtered Klines:** Only 5m closed candles populate the candle buffer (configured via `candle_kline_interval`). All 6 kline intervals (1m, 5m, 15m, 1h, 4h, 1d) still stream for display — only the filter is selective.
- **Trade Journal:** `logs/history.jsonl` — permanent dollar PnL record written per close (one line per trade, decoupled from `--loghfs`).

---

## Log Files

| File | Format | Write Frequency | Contents |
|------|--------|----------------|----------|
| `logs/state.jsonl` | JSONL append | Every 10s | Position state, capital, closed positions |
| `logs/history.jsonl` | JSONL append | Per-trade-close | Permanent trade journal (symbol, pnl_usd, duration, exit reason) |
| `logs/hfscalper_YYYYMMDD_HHMMSS.jsonl` | JSONL append | Per-event (optional) | All trading events (~60 types, only with `--loghfs`) |
| `logs/raw.jsonl` | JSONL append | Per-message | Raw WebSocket data (when `--raw`) |

**State file growth:** ~30.5 MB/hour at 10s interval → ~5 GB/week. Monitor disk usage accordingly.

---

## Known Issues

1. **Margin Type Typo (CROSSED → CROSS):** Sends `"CROSSED"` instead of `"CROSS"` — Binance silently defaults to isolated.
2. **systemctl stop hang:** Pre-existing asyncio signal propagation issue. Workaround: `systemctl kill -s SIGKILL`.
3. **AAVEUSDC qty ≤ 0:** High-priced coins (AAVE ~$67) may fail to meet $5 min notional with small capital. Pre-existing capital constraint.
4. **--loghfs restart:** May need one kill/restart cycle before log files begin writing.
5. **Port race on restart:** Bot holds port 3000; new instance's UI fails silent if port isn't freed. Verify port free before restart.

---

**Last Updated:** June 21, 2026
**Current Positions:** 22 max concurrent
**Capital:** Variable (capped at $1000 live, 90% utilization)
**Dashboard Port:** 3000 (0.0.0.0), proxied via nginx on rebirthtrader.com
