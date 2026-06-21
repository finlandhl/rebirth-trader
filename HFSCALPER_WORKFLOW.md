# Core Engine Workflow

The HFScalper engine is the heart of Rebirth Trader. It orchestrates market data processing, signal generation, trade execution, and position management. Runs on any device — VPS, laptop, or mobile phone (iOS & Android tested).

## Workflow Summary

**1. Market Data Ingestion**
WebSocket streams deliver real-time price data for all configured symbols. Data is parsed, validated, and cached in memory with zero API polling overhead.

**2. Market Assessment**
For each symbol, the engine periodically evaluates:
- **Trend direction** — based on configurable indicators
- **Volatility** — current market conditions
- **Signal confidence** — combined assessment strength

**3. Entry Decision**
When confidence thresholds are met and capital protection gates pass, a market order is placed on Binance Futures. If the order fails due to exchange constraints, the engine automatically rescales and retries.

**4. Position Monitoring**
Each position runs as an independent monitoring coroutine. The engine tracks price, PnL, position age, and trend alignment in real-time.

**5. Exit Execution**
Multiple exit logics are evaluated in priority order on each monitoring cycle. When any logic triggers, the position is closed via Binance API and the result is recorded.

**6. State Persistence**
Active positions are periodically saved to disk. On restart, the engine restores all positions and resumes monitoring automatically.

## Key Characteristics

- **Fully asynchronous** — all operations run concurrently without blocking
- **Fault-isolated** — each position is independently error-handled
- **Configurable** — all thresholds, limits, and behaviors adjustable via configuration
- **Headless-capable** — designed for 24/7 automated operation
