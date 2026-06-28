# Features & Enhancements — v3.0.3

## Cross-Platform Support

Runs on any device with a Linux environment or CLI terminal — VPS servers, laptops, and mobile phones. Tested on iOS (iSH, Termius) and Android (Termux). Full trading capability from a phone.

## State Persistence

The system continuously saves its state to disk every **20 seconds**, allowing full recovery after restarts or crashes. All active positions, configuration, leaderboard ranking, and tracking data are preserved and seamlessly restored when the bot resumes — no trades lost, no manual intervention needed.

## Systemd Integration

One-command installation as a systemd service. The bot survives SSH disconnects, crashes, and server reboots. Automatically restarts with intelligent backoff. Fully headless operation with no interactive prompts required. **LimitNOFILE=65536** applied via systemd override for heavy WebSocket loads.

## Multi-Layered Exit Logic

Six independent exit strategies operating in configurable priority:

- **Loss Recovery Trail** — Adaptive trail that activates at 3.5% loss and exits on 1.1% recovery from trough. Candle-gated.
- **Stop Loss** — Hard protection at 4% from entry price
- **Take Profit** — Exit at 6% profit target
- **Trailing Stop** — Activates at 2.5% profit peak, exits on 0.092% pullback. Candle-gated.
- **Time Limit** — Maximum 7-day holding duration safety
- **RSI Reversal Detection** — Price/RSI divergence triggers hedge consideration

## Candle Gate (15m Trend-Aware Exit Suppression)

A runtime mechanism that compares the **15m EMA(12/26) market trend** against the position direction. When the trend aligns with the position, trailing stops and loss recovery trails are suppressed — allowing winning trades to run longer and losing trades more time to recover. When the trend flips against the position, exits activate immediately.

Requires ≥26 candles (~6.5h) and ≥0.1% EMA spread to produce a valid trend assessment.

## Triple-Filter Entry Signal

No position opens without three-way agreement:
1. **Tick-level trend** — EMA(50/100) from WebSocket price buffer
2. **RSI(14)** — Must be within range and confirm trend direction
3. **Kline confirmation** — 15m EMA(12/26) must agree with tick trend

If any filter disagrees, the entry is blocked with a logged reason.

## Loss Recovery Trail

Embedded within the stop-loss logic: instead of immediately exiting at the hard stop, the bot first checks whether the position has recovered meaningfully from its worst loss. If recovery exceeds 1.1% from the trough, the position is closed at a better price. The hard stop at 4% remains as a safety net.

## Position-Centric Architecture

Each trade runs in its own independent async coroutine with isolated error handling. A failure in one position never affects others. Configurable 20-position concurrency limit (hedge positions excluded).

## Zero-API Price Monitoring

All price data is cached from WebSocket streams. The bot never polls Binance for price updates during position monitoring, resulting in sub-second reaction times with zero API overhead.

## Real-Time Dashboard

Embedded web UI (port 8080) accessible via browser:

- Live wallet balance, realized P&L, best/worst trade
- Interactive equity curve chart (incremental file reads)
- **Lifecycle bar** — position age, profit/loss/trail status
- Position details with entry price, P&L, duration, direction
- Trade history with exit reasons from history.jsonl
- Streak dots — visual trade streak history (last 100)

## Multi-Account Copy Trading (Implemented)

Real-time trade mirroring across multiple Binance Futures accounts from a single master instance. When the master bot opens, closes, or adjusts a position, the same action is relayed to all configured follower accounts simultaneously.

- **Master-slave architecture** — One primary trading instance broadcasts actions to N follower accounts
- **Action relay** — Entry, exit, hedge, and trail events are mirrored with configurable delay and offset
- **Per-account risk scaling** — Each follower can apply its own leverage and position size multiplier relative to the master
- **Independent safety** — Follower accounts maintain their own stop-loss, capital ceiling, and symbol blacklist
- **Lightweight** — Followers require only API credentials and risk config via `users.json` — no full engine instance, no WebSocket streams, no position monitors
- **Heterogeneous support** — Mix accounts with different Binance sub-account tiers, margin modes, and leverage limits

## Leaderboard Entry Priority

When `--leaderboard` is enabled, symbols are ranked by cumulative PnL from `history.jsonl`. Entry signals are processed in order of best-performing symbols first. High-confidence signals (confidence > 0.85) bypass the queue.

## Cloudflare Tunnel Integration

One-command public internet exposure via Cloudflare Tunnel. Auto-installs cloudflared if missing. The dashboard becomes accessible from anywhere with a single command flag (`--tunnel`).

## Defensive Hedge Management

Automatic defensive hedging: when a position suffers consecutive losses for ≥30 minutes and the 15m candle trend confirms the opposite direction, the bot opens a hedge in the opposite direction to offset further downside. If the trend matches the position direction, the hedge is deferred.

## Capital Protection

Multiple configurable safeguards: per-position exposure limits (2.9%), total capital usage caps (90%), minimum notional buffers ($5), maximum balance thresholds ($1000 cap), and leverage tiers (10x/20x/30x based on confidence).

## Trading Pause Window

Configurable quiet hours (23:00–02:00 UTC) during which new entries and hedges are blocked. Active positions continue monitoring normally. Automatic resume when the window ends.

## Symbol Blacklist

Auto-suspension of trading on symbols that show 3 consecutive losses. Configurable 24h cooldown (overridable via `--blacked <symbol> "5H"`). Persists across restarts. Supports pre-blacklisting on startup.

## Order Failure Recovery

If an order fails due to minimum notional constraints, the bot automatically rescales the quantity and retries once. Capital allocation is properly tracked regardless of order success or failure.

## Monitor Exit Retry & Zombie Detection

If a `close_position()` call fails, the monitor retries every 0.5s for up to 30s. After 10 consecutive failures, the bot queries Binance's actual position data. If the position no longer exists on exchange → force-removed as zombie. If it exists → retries with direct market order.

## WebSocket Health Monitoring

Stream health is continuously validated via `last_real_price_update` tracking. Positions cannot be opened on stale or unhealthy data feeds. Configurable staleness thresholds prevent trading on dead streams.

## Binance API Compliance

Full compliance with Binance's post-April 2026 WebSocket routing architecture (public/market/private endpoints). Supports all data types — depth, ticker, kline, and aggregate trade streams — with graceful fallback across different connection environments.

## Trade Journal (history.jsonl)

Every closed trade is appended to `logs/history.jsonl` with: symbol, position_id, direction, entry/exit price, realized_pnl (dollar value), pnl_usd, fees, leverage, exit_reason, exit_time. Used for leaderboard ranking recovery and permanent trade audit trail.

## Command-Line Interface

All features accessible via CLI flags. Comprehensive options for symbol selection (specific or all pairs), persistence mode, automation mode, dashboard enabling, tunnel setup, logging controls, leaderboard, and symbol blacklisting.

---

## Roadmap

Planned enhancements for future releases — priorities driven by buyer demand and market opportunity.

### In Development

| Enhancement | Description |
|-------------|-------------|
| **Dual-Layer Candle Gate** | Macro (15m) for entry authority + micro (5m) for exit authority + both required for hedges |

### Planned

| Enhancement | Description |
|-------------|-------------|
| **Forex, Stocks & Options** | Multi-asset support beyond crypto futures — connect to traditional brokers and trade forex pairs, equities, and options through a unified engine interface |
| **MetaTrader Integration** | Direct MT4/MT5 bridge — feed Rebirth signals into MetaTrader platforms for execution on supported brokers |
| **Partnered Brokers** | Pre-integrated broker partnerships with optimized API adapters, reduced latency routing, and preferential fee structures |
| **Machine Learning Signals** | ML-based signal generation layer — train models on historical and real-time data to generate entry/exit confidence scores |
| **Advanced AI Integration** | LLM-powered market analysis, natural language strategy configuration, and adaptive parameter tuning |
| **Official iOS & Android Application** | Native mobile app with real-time monitoring, push notifications, and one-tap deployment |

---

*Rebirth Trader v3.0.3 — © 2026*
