# Rebirth Trader v3.0.3

**High-Frequency Scalping and Day Trading Bot for Binance Futures**

[![License](https://img.shields.io/badge/license-All%20Rights%20Reserved-red)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.13%2B-blue)](https://www.python.org/)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Android%20%7C%20iOS-lightgrey)
![Status](https://img.shields.io/badge/status-production%20ready-success)
![asyncio](https://img.shields.io/badge/asyncio-concurrent-7B4F9E)
![WebSocket](https://img.shields.io/badge/WebSocket-streaming-4B89DC)
![uv](https://img.shields.io/badge/uv-package%20manager-FFD43B)
![JSONL](https://img.shields.io/badge/JSONL-persistence-2EA44F)
![Cloudflare](https://img.shields.io/badge/Cloudflare-tunnel-F38020)

> At its core, a **scalping bot** — designed to enter and exit trades fast, accumulating small wins across hundreds of markets. The same engine can be used for **day trading** as well, holding positions across larger timeframes. The exit system, the monitoring, the risk management — it all scales from minutes to hours.

## See It Live

A fully operational instance is running at [**rebirthtrader.com**](https://rebirthtrader.com). Open it in any browser — watch the bot trading in real time with live P&L, position tracking, and equity curve.

No login required. No setup. Just open and watch.

For accurate monitoring including active positions, use your **Binance web interface** or **mobile app** — they reflect real-time exchange data directly.

From a mobile phone, open the URL in Chrome or Safari. The dashboard is fully responsive and works on any screen size.

| [![Web dashboard](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-stats.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-stats.jpg) | [![Positions overview](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-positions.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-positions.jpg) |
|:---:|:---:|
| [![Mobile view](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-mobile.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-mobile.jpg) | [![Running on Android via Termux](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-phone.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-phone.mp4) |

*Live dashboard of the original Rebirth Trader running on real markets — accessible from any browser.*

---

## Key Capabilities

| Capability | Description |
|-----------|-------------|
| **Multi-Symbol Streaming** | Simultaneously monitors all USDT + USDC perpetual pairs via WebSocket (~500+ pairs) |
| **Concurrent Position Management** | Per-position coroutines with configurable 20-position concurrency limit |
| **6-Layer Exit Logic** | Loss recovery trail → stop loss → take profit → trailing stop → time limit → RSI reversal detection, all checked every second |
| **Candle Gate (15m EMA 12/26)** | Trend-aware trail suppression — lets winners run and losers recover when the 15m candle trend aligns with the position |
| **Loss Recovery Trail** | Adaptive trail that activates at 3.5% loss and exits on 1.1% recovery from trough |
| **Multi-Account Copy Trading** | Master-slave trade mirroring across N Binance Futures accounts with per-account risk scaling |
| **Leaderboard Entry Priority** | PnL-based symbol ranking — best performers get priority for new entries |
| **Real-Time Dashboard** | Embedded web UI showing live P&L, equity curve, lifecycle bars, and positions |
| **State Persistence** | Full save every 20s with crash-safe JSONL — recover from any crash |
| **Automated Operation** | Systemd integration (servers) with LimitNOFILE=65536 |
| **Symbol Blacklist** | Auto-suspension after 3 consecutive losses for 24h (configurable) |
| **Hedge Mode** | Dual-direction position support with candle-confirmed defensive hedge |
| **Triple-Filter Entry** | Tick EMA 50/100 + RSI(14) + 15m kline EMA 12/26 — all three must agree |
| **Trading Pause** | Blocks new entries 23:00–02:00 UTC — existing positions keep monitoring |
| **Cross-Platform** | Tested on Linux, Android (Termux), iOS (iSH/Termius) |

---

## Why This Stands Out

**Zero API polling** — The bot caches all price data from WebSocket streams. No REST API calls for monitoring. Sub-10ms price reads.

**Six exit layers** — Loss recovery trail, stop loss, take profit, trailing stop (candle-enhanced), time limit, and RSI reversal detection. Every position is checked every second.

**Trend-aware candle gate** — The 15m EMA(12/26) trend analysis prevents premature exits. When the macro trend supports your position, trailing stops and loss trails are suppressed. When the trend flips, exits activate immediately.

**Triple-filter entry** — Tick EMA 50/100 trend must agree with RSI(14) range, and both must agree with the 15m kline EMA(12/26). No signal gets through without full consensus.

**Runs on anything** — VPS, laptop, or $150 Android phone. Full trading capability from a device that fits in your pocket.

**Crash-proof** — State is saved every 20 seconds. Restart the bot and it resumes exactly where it left off, all positions restored.

**Fault isolated** — Each trade runs in its own independent coroutine. One position failing never affects the others.

**Real capital, real markets** — Every component was developed and refined against live market data with actual funds. Not a backtest. Not paper trading. The live performance and trading history is accessible on the Dashboard for independent audits.

**All parameters can be tweaked** — Risk per trade, leverage, exit thresholds, trading hours, markets — everything is configurable depending on your trader profile. For non-developers, use **Hermes Agent** or a similar AI to adjust parameters with natural language instructions.

---

## Built With

| Tool | Purpose |
|------|---------|
| **Python 3.13** | Core runtime |
| **asyncio** | Async concurrency — 20+ parallel positions |
| **Binance WebSocket Streams** | Zero-API real-time price feed (ticker, aggTrade, depth@100ms, klines ×6 intervals) |
| **uv** | Python package & project management |
| **JSONL** | Crash-safe append-only state persistence |
| **Cloudflare Tunnel** | Secure remote dashboard access |
| **Hermes Agent (Nous Research)** | AI agent orchestration & automation |
| **Claude Code (Anthropic)** | AI pair programming for complex features |

---

## Documentation

| Document | Description |
|----------|-------------|
| [docs/README.md](docs/README.md) | Complete system architecture, CLI flags, risk params, exit logic, and operations |
| [docs/live_trading_downloaded.md](docs/live_trading_downloaded.md) | Setup, deployment, and operational guide |
| [FEATURES_ENHANCEMENT.md](FEATURES_ENHANCEMENT.md) | Complete feature catalog and technical capabilities |
| [LIVE_TRADING.md](LIVE_TRADING.md) | Production deployment guide |
| [PERFORMANCE.md](PERFORMANCE.md) | Live trading metrics — updated automatically |

---

## BUY SOURCE

**$1,600 + live realized P&L** — 6+ months of development refined against real capital in live markets. Not a script. A complete production trading infrastructure.

Buy the source code, tweak every parameter, enhance it the way you want — all at your own risk. The bot is designed to be hackable: risk per trade, leverage, exit thresholds, trading hours, markets, everything is configurable.

**The original Rebirth Trader** is available at [rebirthtrader.com](https://rebirthtrader.com) only. It is continuously tested and enhanced with new features, running on real funds in real markets — every trade is verifiable on the live Dashboard.

**White Label buyers** get lifetime upgrades, dedicated support, and rebranding/reselling rights — contact for details.

> **To inquire:** Open an issue on this repository or visit the [live demo website](https://rebirthtrader.com)

**Limited-time source access** is available for early buyers. After the promotional window closes, the source will only be available to White Label buyers.

---

⚠ **Risk Warning:** Trading cryptocurrencies carries significant financial risk. Past performance does not guarantee future results. This software is provided for educational and research purposes. Use at your own risk.

**Rebirth Trader v3.0.3 — © 2026 — All rights reserved.**
