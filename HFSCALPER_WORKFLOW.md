# HFScalper Workflow (Superseded)

**This document has been superseded by [ARCHITECTURE.md](./ARCHITECTURE.md)**

Please refer to `docs/ARCHITECTURE.md` for the complete, up-to-date system architecture reference covering:

- System overview & data flow
- Entry flow & initialization (all current parameters)
- Stream data processing & price buffer
- Signal generation & market assessment
- Full position lifecycle (placeholder → order → position → monitor → close)
- Position sizing & capital management
- 6-layer exit logic system with candle-enhanced trail
- Hedge management (defensive hedge with candle gate)
- Trading pause window (20:00–04:00 UTC blocking)
- Symbol blacklist (`--blacked`) with auto-suspend after 3 consecutive losses
- State persistence (save/restore cycle)
- Signal handler & shutdown sequence (with auto-mode patches)
- Auto-mode & systemd integration
- Error handling & recovery

**Last updated:** June 14, 2026
**Superseded by:** `docs/ARCHITECTURE.md`
