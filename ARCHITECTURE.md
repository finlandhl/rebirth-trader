# Rebirth Trader — System Architecture

## Overview

Rebirth Trader is a terminal-based automated trading system for Binance Futures. It runs on any device with a Linux environment or terminal — servers, laptops, and mobile phones (tested on iOS and Android). It connects via WebSocket streams, executes trades based on configurable entry signals, and manages each position independently for fault isolation.

## Core Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Stream Engine│────▶│  Core Engine  │────▶│  Exit Logic  │
│  (WebSocket) │     │  (Strategy)  │     │  (Multiple)  │
└──────────────┘     └──────────────┘     └──────────────┘
       │                     │                     │
       ▼                     ▼                     ▼
┌──────────────────────────────────────────────────────┐
│                 Binance Futures API                   │
└──────────────────────────────────────────────────────┘
```

## Key Design Principles

**Position-Centric Architecture:** Each trade runs in its own independent async coroutine. A failure in one position never affects others. A configurable concurrency limit prevents overexposure.

**Buffer-Based Price Feed:** All price data is cached in memory from WebSocket streams. The system makes zero API calls for monitoring prices. Buffers include configurable staleness detection to ensure data freshness.

**0 API Call Monitoring:** The bot never polls Binance for price updates during position monitoring. All data arrives via WebSocket and is cached locally, resulting in sub-second reaction times.

**Trend-Aware Exit Suppression:** A trend confirmation mechanism prevents premature exits when the broader market direction still favors the open position, allowing winning trades to run longer.

## Multi-Layered Exit System

The system uses multiple configurable exit strategies operating in priority order:

- **Stop Loss** — Hard protection at a configurable distance from entry
- **Loss Recovery Trail** — Adaptive trail that activates beyond a configurable loss threshold
- **Take Profit** — Configurable profit target
- **Trailing Stop** — Tracks peak profit and exits on configurable pullback
- **Time Limit** — Maximum holding duration safety

## Position Lifecycle

```
Idle → Order Placed → Active (monitoring) → Closed
```

Each position passes through a structured lifecycle with proper state tracking, error isolation, and persistence support.

## Real-Time Dashboard

An embedded web UI provides live monitoring:

- **Live aggregated stats** — Wallet balance, P&L, win rate, active positions
- **Equity curve** — Historical balance tracking with interactive chart
- **Position details** — Active positions with entry price, P&L, and duration
- **Trade history** — Closed trades with exit reasons and performance
- **Internationalization** — Multi-language interface support

## Automated Operation

- **State Persistence:** Full save and recovery of positions, config, and blacklist across restarts
- **Systemd Integration:** Runs as a service with automatic startup on boot
- **Trading Pause Window:** Configurable quiet hours with automatic resume
- **Symbol Blacklist:** Auto-suspension after configurable consecutive losses

## Capital Protection

The system enforces multiple configurable capital safeguards: per-position exposure limits, total usage caps, minimum notional buffers, and maximum balance thresholds.

For complete feature details, see [FEATURES_ENHANCEMENT.md](FEATURES_ENHANCEMENT.md).
