# Rebirth Trader v3.0.0

**High-Frequency Scalping Bot for Binance Futures**

[![License](https://img.shields.io/badge/license-All%20Rights%20Reserved-red)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.13%2B-blue)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Android%20%7C%20iOS-lightgrey)]()
[![Status](https://img.shields.io/badge/status-production%20ready-success)]()

> A terminal-based, fully automated trading system engineered for 24/7 operation on any device with a Linux environment or CLI terminal — from VPS servers to mobile phones. Tested on iOS (iSH, Termius) and Android (Termux).

---

## Demo

*A live instance is running and viewable at [72.61.160.72](http://72.61.160.72)*

<p align="center">
  <em>Dashboard screenshot will appear here — capture from live instance</em>
</p>

---

## Quick Start

```bash
# Clone the repo
git clone https://github.com/finlandhl/rebirth-trader.git
cd rebirth-trader

# Install dependencies
uv sync

# Run in simulation mode (no real trading)
uv run streamer.py --scalp --all

# Run live with dashboard
uv run streamer.py --scalp --live --all --persist --ui
```

That's it. One command to deploy. The bot handles everything else — WebSocket connections, position management, state persistence, and dashboard hosting.

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

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=finlandhl/rebirth-trader&type=Date)](https://star-history.com/#finlandhl/rebirth-trader&Date)

---

⚠ **Risk Warning:** Trading cryptocurrencies carries significant financial risk. Past performance does not guarantee future results. This software is provided for educational and research purposes. Use at your own risk.

**Rebirth Trader v3.0.0 — © 2026 — All rights reserved.**
