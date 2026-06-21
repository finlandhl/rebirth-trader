# Features & Enhancements — v3.0.0

## Cross-Platform Support

Runs on any device with a Linux environment or CLI terminal — VPS servers, laptops, and mobile phones. Tested on iOS (iSH, Termius) and Android (Termux). Full trading capability from a phone.

## State Persistence

The system continuously saves its state to disk, allowing full recovery after restarts or crashes. All active positions, configuration, and tracking data are preserved and seamlessly restored when the bot resumes — no trades lost, no manual intervention needed.

## Systemd Integration

One-command installation as a systemd service. The bot survives SSH disconnects, crashes, and server reboots. Automatically restarts with intelligent backoff. Fully headless operation with no interactive prompts required.

## Multi-Layered Exit Logic

Five independent exit strategies operating in configurable priority:

- **Stop Loss** — Hard protection at a configurable distance from entry price
- **Loss Recovery Trail** — Adaptive trail that activates beyond a configurable loss threshold, exiting when price recovers from its worst point
- **Take Profit** — Exit at configurable profit target
- **Trailing Stop** — Tracks peak profit and exits on configurable pullback
- **Time Limit** — Maximum holding duration safety

## Trend-Aware Exit Suppression (Candle Gate)

A runtime mechanism that compares the broader market trend against the position direction. When the trend aligns with the position, certain exit methods are delayed, allowing winning trades to run longer and losing trades more time to recover. This prevents premature exits during favorable market conditions.

## Loss Recovery Trail

Embedded within the stop-loss logic: instead of immediately exiting at the hard stop, the bot first checks whether the position has recovered meaningfully from its worst loss. If it has, the position is closed at a better price. The hard stop remains as a safety net.

## Position-Centric Architecture

Each trade runs in its own independent async coroutine with isolated error handling. A failure in one position never affects others. Configurable concurrency limit prevents overexposure.

## Zero-API Price Monitoring

All price data is cached from WebSocket streams. The bot never polls Binance for price updates during position monitoring, resulting in sub-second reaction times with zero API overhead.

## Real-Time Dashboard

Embedded web UI accessible via browser:

- Live wallet balance, P&L, win rate, and active positions
- Interactive equity curve chart
- Position details with entry price, P&L, and duration
- Trade history with exit reasons
- Multi-language support

## Cloudflare Tunnel Integration

One-command public internet exposure via Cloudflare Tunnel. The dashboard becomes accessible from anywhere with a single command flag.

## Defensive Hedge Management

Automatic defensive hedging: when a position suffers consecutive losses and the market trend still opposes it, the bot can open a hedge in the opposite direction to offset further downside.

## Capital Protection

Multiple configurable safeguards: per-position exposure limits, total capital usage caps, minimum notional buffers, and maximum balance thresholds. All enforced in real-time before each trade.

## Trading Pause Window

Configurable quiet hours during which new entries and hedges are blocked. Active positions continue monitoring normally. Automatic resume when the window ends.

## Symbol Blacklist

Auto-suspension of trading on symbols that show repeated consecutive losses. Configurable loss threshold and cooldown duration. Persists across restarts. Supports pre-blacklisting and cooldown overrides via CLI.

## Order Failure Recovery

If an order fails due to minimum notional constraints, the bot automatically rescales the quantity and retries once. Capital allocation is properly tracked regardless of order success or failure.

## WebSocket Health Monitoring

Stream health is continuously validated. Positions cannot be opened on stale or unhealthy data feeds. Configurable staleness thresholds prevent trading on dead streams.

## Binance API Compliance

Full compliance with Binance's WebSocket routing architecture. Supports all data types — depth, ticker, kline, and aggregate trade streams — with graceful fallback across different connection environments.

## Command-Line Interface

All features accessible via CLI flags. Comprehensive options for symbol selection (specific or all pairs), persistence mode, automation mode, dashboard enabling, tunnel setup, logging controls, and symbol blacklisting.
