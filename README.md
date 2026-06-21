# Rebirth Trader v3.0.0

**High-Frequency Scalping Bot for Binance Futures**

Terminal-based, 24/7 automated trading bot with advanced exit strategies, real-time dashboard, and multi-symbol support.

## Features

- **HFScalper Engine** — High-frequency scalping with per-position async coroutines
- **5 Exit Logics** — Trailing Stop, Stop Loss, Take Profit, Loss Recovery Trail, Time Limit
- **Real-time Dashboard** — Live P&L, equity curve, position monitoring via web UI
- **Multi-Symbol Support** — Trade all USDT + USDC perpetual pairs simultaneously
- **24/7 Operation** — Systemd service with automatic recovery
- **Terminal-Based** — Zero GUI dependencies, runs on any Linux VPS

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Stream Engine│────▶│  HFScalper   │────▶│  Exit Logic  │
│  (WebSocket) │     │  (Strategy)  │     │  (5 methods) │
└──────────────┘     └──────────────┘     └──────────────┘
       │                     │                     │
       ▼                     ▼                     ▼
┌──────────────────────────────────────────────────────┐
│                 Binance Futures API                   │
└──────────────────────────────────────────────────────┘
```

## Live Demo

Dashboard: [72.61.160.72](http://72.61.160.72)

## Documentation

- [Architecture](ARCHITECTURE.md)
- [Features & Enhancements](FEATURES_ENHANCEMENT.md)
- [HFScalper Workflow](HFSCALPER_WORKFLOW.md)
- [Live Trading Guide](LIVE_TRADING.md)

## BUY SOURCE

Contact: **finland.lasme@gmail.com**

Full codebase with 30 days of updates. Ready to deploy on your laptop, server, or mobile.

---

⚠ **Risk Warning:** Trading cryptocurrencies carries significant financial risk. Past performance does not guarantee future results. Use at your own risk.

Rebirth Trader v3.0.0 — © 2026 — All rights reserved.
