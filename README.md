# Rebirth Trader v3.0.0

**High-Frequency Scalping Bot for Binance Futures**

[![License](https://img.shields.io/badge/license-All%20Rights%20Reserved-red)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.13%2B-blue)](https://www.python.org/)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Android%20%7C%20iOS-lightgrey)
![Status](https://img.shields.io/badge/status-production%20ready-success)
![asyncio](https://img.shields.io/badge/asyncio-concurrent-7B4F9E)
![WebSocket](https://img.shields.io/badge/WebSocket-streaming-4B89DC)
![uv](https://img.shields.io/badge/uv-package%20manager-FFD43B)
![JSONL](https://img.shields.io/badge/JSONL-persistence-2EA44F)
![Cloudflare](https://img.shields.io/badge/Cloudflare-tunnel-F38020)
![Selenium](https://img.shields.io/badge/Selenium-screenshots-43B02A)

> A terminal-based, fully automated trading system engineered for 24/7 operation on any device with a Linux environment or CLI terminal — from VPS servers to mobile phones. Developed and refined with real capital across live market conditions. Tested on iOS (iSH, Termius) and Android (Termux).

## See It Live

A fully operational instance is running at [**72.61.160.72**](http://72.61.160.72). Open it in any browser — watch the bot trading in real time with live P&L, position tracking, and equity curve.

No login required. No setup. Just open and watch.

From a mobile phone, open the URL in Chrome or Safari. The dashboard is fully responsive and works on any screen size.

| [![Web dashboard](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-stats.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-stats.jpg) | [![Positions overview](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-positions.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-positions.jpg) |
|:---:|:---:|
| [![Mobile view](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-mobile.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-mobile.jpg) | [![Running on Android via Termux](https://raw.githubusercontent.com/finlandhl/rebirth-trader/main/assets/demo-phone.jpg)](https://github.com/finlandhl/rebirth-trader/blob/main/assets/demo-phone.mp4) |

*embedded dashboard of bot running accessible through your browser*

---

## Key Capabilities

| Capability | Description |
|-----------|-------------|
| **Multi-Symbol Streaming** | Simultaneously monitors all USDT + USDC perpetual pairs via WebSocket |
| **Concurrent Position Management** | Per-position coroutines with configurable concurrency limits |
| **Multi-Layered Exit Logic** | Five independent exit strategies operating in priority order |
| **Trend-Aware Trail Suppression** | Reduces premature exits during strong trend-aligned moves |
| **Loss Recovery Trail** | Adaptive trail that activates beyond a configurable loss threshold |
| **Real-Time Dashboard** | Embedded web UI showing live P&L, equity curve, and positions |
| **State Persistence** | Full save and recovery of positions and config across restarts |
| **Automated Operation** | Systemd integration (servers) or background process (mobile) |
| **Symbol Blacklist** | Auto-suspension after consecutive losses |
| **Hedge Management** | Dual-direction position support for correlated pairs |
| **Cross-Platform** | Tested on Linux, Android (Termux), iOS (iSH/Termius) |

---

## Why This Stands Out

**Zero API polling** — The bot caches all price data from WebSocket streams. No REST API calls for monitoring. Sub-10ms price reads.

**Runs on anything** — VPS, laptop, or $150 Android phone. Full trading capability from a device that fits in your pocket.

**Crash-proof** — State is saved every 60 seconds. Restart the bot and it resumes exactly where it left off, all positions restored.

**Fault isolated** — Each trade runs in its own coroutine. One position failing never affects the others.

**Real capital, real markets** — Every component was developed and refined against live market data with actual funds. Not a backtest. Not paper trading.

---

## Built With

| Tool | Purpose |
|------|---------|
| **Python 3.13** | Core runtime |
| **asyncio** | Async concurrency — 20+ parallel positions |
| **Binance WebSocket Streams** | Zero-API real-time price feed |
| **uv** | Python package & project management |
| **JSONL** | Crash-safe append-only state persistence |
| **Cloudflare Tunnel** | Secure remote dashboard access from anywhere |
| **Selenium + Chromium** | Automated screenshot generation for assets |
| **Claude Code (Anthropic)** | AI pair programming for complex features |
| **Hermes Agent (Nous Research)** | AI agent orchestration & automation |
| **Kimi K2.6 (Moonshot AI)** | Reasoning & code generation |
| **DeepSeek** | AI code synthesis & problem solving |

---

## Documentation

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design, component interactions, and data flow |
| [FEATURES_ENHANCEMENT.md](FEATURES_ENHANCEMENT.md) | Complete feature catalog and technical capabilities |
| [HFSCALPER_WORKFLOW.md](HFSCALPER_WORKFLOW.md) | Core trading engine workflow |
| [LIVE_TRADING.md](LIVE_TRADING.md) | Setup, deployment, and operational guide |

---

## BUY SOURCE

> **Purchase includes:**
>
> ✓ Complete Python source code — all modules, configs, and build tooling
>
> ✓ Full internal documentation — architecture reference with implementation details
>
> ✓ 30 days of updates — latest features and improvements
>
> ✓ Ready to deploy — VPS, laptop, or mobile (iOS & Android supported)
>
> **To inquire:** Open an issue on this repository or visit the [live demo website](http://72.61.160.72)

---

⚠ **Risk Warning:** Trading cryptocurrencies carries significant financial risk. Past performance does not guarantee future results. This software is provided for educational and research purposes. Use at your own risk.

**Rebirth Trader v3.0.0 — © 2026 — All rights reserved.**
