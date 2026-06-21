# Rebirth Trader v3.0.0

**High-Frequency Scalping Bot for Binance Futures**

A terminal-based, fully automated trading system engineered for 24/7 operation on any device with a Linux environment or CLI terminal — from VPS servers to mobile phones. Tested on iOS (iSH, Termius) and Android (Termux). Designed with a focus on capital preservation and consistent execution across hundreds of markets simultaneously.

---

## Overview

Rebirth Trader connects directly to Binance Futures via WebSocket streams, executing trades based on configurable entry signals and a multi-layered exit management system. Every position is monitored as an independent coroutine, allowing the system to manage dozens of concurrent trades across multiple symbols with zero API polling overhead.

The system runs as a systemd service (on servers) or as a background process (on mobile via Termux/iSH), persists its state across restarts, and provides a real-time web dashboard for monitoring performance, positions, and equity.

---

## Key Capabilities

| Capability | Description |
|-----------|-------------|
| **Multi-Symbol Streaming** | Simultaneously monitors all USDT and USDC perpetual pairs via WebSocket |
| **Concurrent Position Management** | Per-position coroutines with configurable concurrency limits |
| **Multi-Layered Exit Logic** | Five independent exit strategies operating in priority order |
| **Trend-Aware Trail Suppression** | Reduces premature exits on strong trend-aligned moves |
| **Loss Recovery Trail** | Adaptive trail that activates beyond a configurable loss threshold |
| **Real-Time Dashboard** | Embedded web UI showing live P&L, equity curve, and position details |
| **State Persistence** | Full save and recovery of positions and config across restarts |
| **Automated Operation** | Systemd integration for autonomous 24/7 runtime |
| **Symbol Blacklist** | Auto-suspension after consecutive losses |
| **Hedge Management** | Dual-direction position support for correlated pairs |
| **Trading Pause Window** | Configurable quiet hours with automatic resume |
| **Capital Protection** | Per-position, total-usage, and balance caps |

---

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Stream Engine│────▶│  Core Engine  │────▶│  Exit Logic  │
│  (WebSocket) │     │  (Strategy)  │     │  (5 methods) │
└──────────────┘     └──────────────┘     └──────────────┘
       │                     │                     │
       ▼                     ▼                     ▼
┌──────────────────────────────────────────────────────┐
│                 Binance Futures API                   │
└──────────────────────────────────────────────────────┘
```

For detailed system architecture, see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Live Demo

A live instance is running and viewable at: [72.61.160.72](http://72.61.160.72)

*Note: This demo shows real trading activity in progress.*

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

Purchase includes:

- **Complete source code** — All Python modules, configuration files, and build tooling
- **Full documentation** — Internal architecture reference with implementation details
- **30 days of updates** — Latest features and improvements
- **Ready to deploy** — Works on any Linux VPS, server, laptop, or mobile phone (iOS & Android supported)

**To inquire:** Open an issue on this repository or contact via the [live demo website](http://72.61.160.72).

---

⚠ **Risk Warning:** Trading cryptocurrencies carries significant financial risk. Past performance does not guarantee future results. This software is provided for educational and research purposes. Use at your own risk.

Rebirth Trader v3.0.0 — © 2026 — All rights reserved.
