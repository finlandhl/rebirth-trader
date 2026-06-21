# HFScalper Workflow

**Status:** Superseded — this document has been replaced by the system architecture reference.

## Reference

For a complete, up-to-date overview of the system architecture, please see the [Architecture Guide](./ARCHITECTURE.md).

The Architecture Guide covers:

- **System overview & data flow** — End-to-end pipeline from data ingestion to trade execution
- **Entry flow & initialization** — Parameter configuration and startup sequence
- **Stream data processing & price buffer** — Real-time market data handling
- **Signal generation & market assessment** — Entry signal evaluation logic
- **Position lifecycle management** — Full trade lifecycle from placeholder through monitoring to close
- **Position sizing & capital management** — Risk-based allocation strategies
- **Exit logic system** — Multi-layer stop management with trailing mechanisms
- **Hedge management** — Defensive hedging with market condition gating
- **Trading schedules** — Configurable trading windows and pause periods
- **Symbol management** — Blacklist and auto-suspension after consecutive losses
- **State persistence** — Save/restore cycle for crash recovery
- **Shutdown & signal handling** — Graceful shutdown and auto-mode integration
- **Auto-mode & service management** — Headless operation and process supervision
- **Error handling & recovery** — Resilience patterns and failure recovery

---

**Last updated:** June 14, 2026
**Superseded by:** `docs/ARCHITECTURE.md`
