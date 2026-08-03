# Live Performance

*Updated: 2026-08-03 20:56:18 UTC*

Real-time performance metrics from the live trading instance at **[rebirthtrader.com](https://rebirthtrader.com)**.

---

## Snapshot

| Metric | Value |
|--------|-------|
| **Wallet Balance** | $6.51 |
| **Initial Capital** | $10.49 |
| **Realized P&L** | $-3.57 |
| **Unrealized P&L** | $-0.42 |
| **Total P&L** | $-3.99 |
| **Win Rate** | 43.1% |
| **Closed Trades** | 130 |
| **Wins / Losses** | 56W / 74L |
| **Active Positions** | 8 |
| **Avg Win** | $0.20 |
| **Avg Loss** | $-0.18 |
| **Total Fees** | $1.03 |
| **Bot Uptime** | 11h 21m |

---

## Best & Worst Trades

| | Symbol | P&L | Exit Reason |
|---|--------|-----|-------------|
| **Best** | AIOUSDT | $0.43 | Emergency close: 34s after original exit |
| **Worst** | IDOLUSDT | $-0.98 | Stop Loss: -9.98% loss |

---

## Exit Reason Breakdown

| Exit Reason | Count | Net P&L |
|-------------|-------|---------|
| **Time Limit** | 48x | $-1.03 |
| **Trailing Stop** | 32x | $8.06 |
| **Stop Loss** | 20x | $-7.49 |
| **Loss Trail** | 17x | $-2.14 |
| **By Supervisor** | 11x | $0.25 |
| **Emergency Close** | 2x | $0.13 |
| **Total** | 130x | $-2.23 |

## Restart History

| # | Time (UTC) | Reason |
|---|------------|--------|
| 1 | 2026-06-21 08:28:16 UTC | Deployment update — bot restarted with new feature additions |
| 2 | 2026-06-21 12:55:20 UTC | Configuration tweak — risk parameter adjustment |
| 3 | 2026-06-21 13:01:46 UTC | Post-config restart — systemd auto-restart after test cycle |
| 4 | 2026-06-21 13:39:55 UTC | Bug fix patch — corrected exit logic edge case |
| 5 | 2026-06-21 14:01:29 UTC | Performance tuning — stream buffer optimization |
| 6 | 2026-06-21 15:43:16 UTC | WebSocket reconnect — stream health update |
| 7 | 2026-06-21 16:09:22 UTC | Feature update — enhanced trailing stop suppression |
| 8 | 2026-06-21 16:11:54 UTC | Hotfix — symbol blacklist tweak |
| 9 | 2026-06-21 16:24:08 UTC | Bug fix — hedge mode state restoration |
| 10 | 2026-06-21 16:34:47 UTC | Final deployment — current stable instance |

---

> Data refreshes automatically every 12 hours. See the [live dashboard](https://rebirthtrader.com) for real-time updates.


---

### ⚠ Disclaimers

**Educational Purpose Only** — These metrics are provided for informational and educational purposes. They do not constitute financial advice, a recommendation, or a guarantee of future results. Trading cryptocurrency futures carries substantial risk.

**Computation Buffer** — All figures shown are derived from the bot's internal tracking and may differ from Binance's official account statements. Discrepancies can arise from rounding, fee scheduling, funding rate timing, and API response delays. Always verify against your Binance wallet history for absolute accuracy.

*Rebirth Trader v3.0.0 — © 2026*
