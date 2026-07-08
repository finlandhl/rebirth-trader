# Changelog — Rebirth Trader

All notable patches, fixes, and enhancements applied to the live trading bot.

---

## 2026-07-08

- **Sentinel profile enhanced** — 18 new runtime fields added to UserProfile: long/short ratio, average confidence, hedge status, positions in red/green, consecutive loss streak, drawdown %, active symbol count, total trades, win rate, avg PnL per trade. Profile now gives the LLM a richer picture of bot state during decision scans.

- **History.jsonl enriched** — `event: position_closed`, `side`, `entry_time`, clean `exit_reason` (no more raw `str(namedtuple)`), and `logic_name` fields added. Entries are now filterable and machine-readable.

- **Renamed `max_concurrent_positions` → `max_concurrent_symbols`** — 73 references across hfscalper.py and profile.py updated. Name now accurately reflects that the limit is per-symbol (hedge pairs excluded).

- **Ghost detection interval reduced** — first check 3 min, subsequent every 5 min (was 30s/120s). Fixes reconcile rate-limit errors that were flooding logs.

## 2026-07-07

- **Dashboard uPnL sync fixed** — Three bugs patched in server.py: equity chart was mixing ratio values with dollar amounts, summary header was summing ratios instead of USDT, and per-position PnL preferred theoretical `coin_position` over actual `executed_quantity`. All positions now show correct PnL regardless of leverage.

- **Hedge pair PnL fix** — Hedge calculation in `check_exits_and_monitor_positions` now uses `executed_quantity` (actual filled size) instead of `coin_position` (theoretical). No more PnL skips on hedges.

- **Self-healing entry guard deployed** — `_pending_entries` Dict with 30s TTL prevents duplicate order races. A symbol entering the flow is tracked by timestamp; expired entries are cleaned on access. Two entry paths guarded (stream data + leaderboard processor).

- **Duplicate orders root cause fixed** — Master account credentials were accidentally listed in `users.json` under the multi-account executor, causing the bot to submit every order twice (once via main path, once via multi-acc path). `users.json` cleared.

- **Lifecycle color system redesigned** — 7-state scheme (0=green/entry, 1=red/initial, 2=yellow/recovering, 3=orange/loss-trail, 4=dark-green/near-TP, 5=dark-red/near-SL, 6=blue/trail-active) with time-progress tracking.

- **Dashboard lifecycle bar freeze fixed** — Bar was using `lc[-1].t` (last lifecycle timestamp) instead of current time. Changed to `Date.now() - entry_ms` so bar always reflects real elapsed time.

- **Max concurrent symbols increased 5 → 20** — Single line change at `self.max_concurrent_symbols = 20` (was 5). Enables scaling to full portfolio.

## 2026-07-06

- **Position lifecycle color system implemented** — 6-state scheme tracking entry, red, yellow, orange, dark-green, dark-red, blue phases. Colors reflect real-time position health.

- **TimeLimitExit rebuilt** — `max_holding_minutes` made required in `Position` dataclass; single source of truth at `HFScalper.max_holding_minutes`. No more hardcoded defaults.

## 2026-07-05

- **Dashboard equity chart live data** — Continuous equity tracking from bot start; chart now shows wallet + unrealized PnL progression.

## 2026-07-04

- **Multi-account executor initial integration** — users.json schema, copier-only accounts, master account excluded.

## 2026-07-03

- **Entry confidence calibration deployed** — 3×3 decision matrix (TREND × VOLUME). Strong+Strong → HIGH confidence, Weak+Weak → INVALID. Natural separation via tanh + conviction scoring.
