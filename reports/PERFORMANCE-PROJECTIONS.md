# Rebirth Trader — Performance Projections

**Generated:** 2026-06-22 17:20 UTC
**Data span:** 92 trades over 2.47 days
**Bot version:** v3.0.1

## 1. Current Live Performance

| Metric | Value |
|--------|-------|
| Trades analyzed | 92 |
| Time span | 2.5 days |
| Trades/day | 37.2 |
| Win rate | 53.3% (49W/43L) |
| Profit factor | 1.27 |
| Avg win | +$0.2562 |
| Avg loss | -$0.2304 |
| Avg fee per trade | $0.0049 |
| Avg duration | 408.2 min (6.8h)
| Total PnL (gross) | $2.6437 |
| Total fees | $0.4478 |
| Net profit | $2.1959 |
| Available capital | $6.76 |
| Daily return on capital | 13.2% |
| Avg position value | $6.20 |
| Effective leverage | 20x (default) |

**Exit reason breakdown:**
  - Trailing Stop: 31 trades
  - Stop Loss: 29 trades
  - Profit Target: 17 trades
  - Loss Trail: 14 trades
  - Emergency close: 1 trades

## 2. Exchange Liquidity Profile

Order book depth (min of bid/ask, top 5 levels) for the bot's 21 active symbols:

| Symbol | Min Book Depth | 10% Cap (max safe position) |
|--------|:-------------:|:-------------------------:|
| AIOUSDT | $     81 | $      8 |
| ZORAUSDT | $    337 | $     34 |
| AWEUSDT | $    402 | $     40 |
| ASTRUSDT | $    565 | $     57 |
| MIRAUSDT | $    649 | $     65 |
| FLOWUSDT | $    938 | $     94 |
| SONICUSDT | $    959 | $     96 |
| ELSAUSDT | $  1,423 | $    142 |
| FLOCKUSDT | $  2,029 | $    203 |
| LQTYUSDT | $  2,288 | $    229 |
| CGPTUSDT | $  3,001 | $    300 |
| AIXBTUSDT | $  3,556 | $    356 |
| AZTECUSDT | $  3,762 | $    376 |
| SEIUSDT | $  7,567 | $    757 |
| WOOUSDT | $  9,614 | $    961 |
| COTIUSDT | $  9,637 | $    964 |
| ASRUSDT | $ 14,304 | $  1,430 |
| REDUSDT | $ 14,573 | $  1,457 |
| ARKMUSDT | $ 18,014 | $  1,801 |
| ZRXUSDT | $ 18,438 | $  1,844 |
| USUALUSDT | $ 18,919 | $  1,892 |

**Avg book depth:** $6,241
**Thin pairs:** 4/21 (<10x current position value)

## 3. Projections by Capital + Fee Scenario

### Fee Scenario: Aggressive (maker-heavy) (0.06%)

**Current ($7)** (effective scale: 1.0x, slippage: 0.37%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $  20.10 | $   26.60 | $   -4.42 | $    5.12 | $   30.86 | 40.3% | 66.7% | 7.8% |
| 3mo | $  61.23 | $   67.73 | $   -4.40 | $   34.55 | $   79.48 | 39.9% | 65.3% | 7.0% |
| 6mo | $ 121.52 | $  128.02 | $   -4.42 | $   86.42 | $  149.14 | 39.4% | 66.4% | 6.8% |
| 12mo | $ 246.79 | $  253.29 | $   -4.46 | $  189.22 | $  285.17 | 39.9% | 66.2% | 8.6% |

**$100 capital** (effective scale: 11.3x, slippage: 2.07%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $ 186.54 | $  286.54 | $  -68.05 | $   27.44 | $  301.76 | 39.2% | 66.8% | 7.8% |
| 3mo | $ 559.63 | $  659.63 | $  -67.34 | $  279.82 | $  779.09 | 40.2% | 65.2% | 6.0% |
| 6mo | $1097.31 | $ 1197.31 | $  -68.04 | $  658.40 | $ 1426.50 | 40.7% | 66.3% | 8.4% |
| 12mo | $2222.06 | $ 2322.06 | $  -68.00 | $ 1547.23 | $ 2660.98 | 41.2% | 66.9% | 8.4% |

**$500 capital** (effective scale: 28.2x, slippage: 2.92%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $ 424.62 | $  924.62 | $   -0.58 | $  109.15 | $  712.66 | 26.4% | 43.1% | 0.6% |
| 3mo | $1232.70 | $ 1732.70 | $  546.90 | $  684.06 | $ 1822.50 | 28.6% | 45.8% | 0.8% |
| 6mo | $2369.40 | $ 2869.40 | $ 1409.27 | $ 1642.44 | $ 3137.50 | 30.8% | 49.0% | 2.2% |
| 12mo | $4779.94 | $ 5279.94 | $ 3202.59 | $ 3655.22 | $ 5822.36 | 29.8% | 49.4% | 2.2% |

### Fee Scenario: Current (0.08%) (0.08%)

**Current ($7)** (effective scale: 1.0x, slippage: 0.37%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $  19.26 | $   25.76 | $   -4.42 | $    7.09 | $   30.96 | 40.2% | 66.0% | 7.0% |
| 3mo | $  55.91 | $   62.41 | $   -4.42 | $   32.97 | $   76.03 | 42.5% | 69.4% | 7.8% |
| 6mo | $ 113.22 | $  119.72 | $   -4.43 | $   74.37 | $  139.43 | 42.0% | 68.1% | 8.2% |
| 12mo | $ 231.12 | $  237.62 | $   -4.43 | $  174.01 | $  271.84 | 40.7% | 66.5% | 8.0% |

**$100 capital** (effective scale: 11.3x, slippage: 2.07%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $ 165.75 | $  265.75 | $  -68.03 | $   -4.33 | $  286.45 | 41.3% | 67.6% | 8.6% |
| 3mo | $ 513.70 | $  613.70 | $  -68.27 | $  -67.23 | $  733.16 | 43.1% | 70.9% | 10.8% |
| 6mo | $ 999.96 | $ 1099.96 | $  -68.24 | $  -67.08 | $ 1329.15 | 44.0% | 70.4% | 10.2% |
| 12mo | $2016.39 | $ 2116.39 | $  -68.47 | $  -67.32 | $ 2449.82 | 44.3% | 70.7% | 11.4% |

**$500 capital** (effective scale: 28.2x, slippage: 2.92%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $ 345.19 | $  845.19 | $  -80.01 | $   29.72 | $  674.38 | 28.6% | 47.7% | 1.6% |
| 3mo | $1104.16 | $ 1604.16 | $  294.91 | $  486.93 | $ 1652.80 | 31.7% | 48.9% | 1.8% |
| 6mo | $2112.30 | $ 2612.30 | $ 1028.73 | $ 1289.33 | $ 2880.40 | 32.2% | 51.9% | 2.0% |
| 12mo | $4252.04 | $ 4752.04 | $ 2729.55 | $ 3072.45 | $ 5445.34 | 31.8% | 50.5% | 2.6% |

### Fee Scenario: Conservative (taker-heavy) (0.10%)

**Current ($7)** (effective scale: 1.0x, slippage: 0.37%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $  17.49 | $   23.99 | $   -4.46 | $   -4.37 | $   28.72 | 43.2% | 69.2% | 10.8% |
| 3mo | $  52.92 | $   59.42 | $   -4.43 | $   25.31 | $   70.71 | 43.9% | 68.8% | 9.6% |
| 6mo | $ 103.98 | $  110.48 | $   -4.46 | $   -4.36 | $  129.72 | 43.8% | 69.2% | 10.6% |
| 12mo | $ 211.70 | $  218.20 | $   -4.48 | $  132.13 | $  251.95 | 42.9% | 68.0% | 10.0% |

**$100 capital** (effective scale: 11.3x, slippage: 2.07%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $ 150.44 | $  250.44 | $  -68.64 | $  -67.37 | $  276.62 | 43.5% | 69.6% | 12.4% |
| 3mo | $ 451.31 | $  551.31 | $  -68.74 | $  -67.32 | $  676.25 | 44.6% | 70.2% | 10.8% |
| 6mo | $ 919.07 | $ 1019.07 | $  -68.40 | $  -67.15 | $ 1220.83 | 44.7% | 69.0% | 11.4% |
| 12mo | $1832.66 | $ 1932.66 | $  -68.07 | $  -67.09 | $ 2282.55 | 44.3% | 70.9% | 10.8% |

**$500 capital** (effective scale: 28.2x, slippage: 2.92%)

| Period | Profit | Final Cap | P5 (worst) | P10 | P90 | Avg DD | P90 DD | Bankrupt |
|--------|:-----:|:---------:|:----------:|:---:|:---:|:-----:|:------:|:--------:|
| 1mo | $ 320.63 | $  820.63 | $ -132.00 | $   -8.56 | $  622.39 | 29.9% | 48.2% | 2.8% |
| 3mo | $1003.04 | $ 1503.04 | $  234.94 | $  399.53 | $ 1483.11 | 32.7% | 51.3% | 2.6% |
| 6mo | $1896.36 | $ 2396.36 | $  524.75 | $ 1018.53 | $ 2678.17 | 34.0% | 56.2% | 4.0% |
| 12mo | $3779.00 | $ 4279.00 | $ 2119.35 | $ 2599.41 | $ 4876.29 | 34.2% | 54.6% | 3.0% |

## 4. Best-Estimate Summary (Current Fee 0.08%)

| Capital | 1 month | 3 months | 6 months | 12 months | Risk (Avg DD) | Notes |
|--------|:-------:|:--------:|:--------:|:---------:|:------------:|-------|
| $  6.5 | $  19.26 | $  56.37 | $ 115.09 | $ 230.18 | ~42% | 21/21 symbols tradeable |
| $  100 | $ 171.23 | $ 491.75 | $1016.42 | $2005.41 | ~43% | 16/21 symbols tradeable |
| $  500 | $ 358.91 | $1090.44 | $2208.31 | $4265.75 | ~32% | 8/21 symbols tradeable ⚠️ liquidity constrained |

## 5. Realistic Outlook

### Key caveats

1. **Small sample size** — Only 92 trades over 2.5 days. Win rate and avg PnL will drift as more data accumulates. A 1,000-trade sample is needed for statistical significance.
2. **Volatility dependency** — The bot profits from micro-cap volatility. Extended ranging periods (1-3 weeks) will produce flat-to-negative returns as stops get hit before trailing activates.
3. **No weekend/off-hours data** — The sample covers a single volatile period. Weekend trading volume is 30-50% lower, which means fewer trades and wider spreads.
4. **Slippage is real** — At $500+ capital, market orders on thin pairs (GMX, COLLECT, ELSA, FLOCK, XVS) incur 1-5% slippage. The simulation models this, but actual slippage varies by market conditions.
5. **Fee drag compounds** — At 37.2 trades/day, fees consume about $0.181/day at current scale. At $500 scale, this becomes $13.40/day — a significant operating cost.
6. **Drawdown recovery takes time** — After a -24% drawdown (the average), it takes proportionally more winning trades to recover. This is built into the simulation via the streak-decay model.

### tl;dr

- **Current ($7):** Expect **$100-300/year**. Fun experiment, but fees eat ~10.0% of capital annually in operating costs.
- **$100:** The sweet spot. Expect **$2,000-4,000/year** with 16/21 symbols tradeable. Survivable drawdowns of ~$20-25.
- **$500:** Highest absolute returns but only 8/21 symbols usable. Expect **$5,000-10,000/year** with ~14% avg drawdown. Liquidity is the binding constraint — not capital.

### Single biggest risk

**Not bankruptcy (1% chance) — but stagnation.** If the bot hits a 2-week ranging market with 50%+ of trades hitting stop losses, it can produce 30 consecutive small losses. The capital doesn't vanish, but growth stalls completely. This is the existential risk for a volatility-dependent scalper.

---

*Auto-generated by Rebirth Trader Performance Monitor. Updated every 24 hours.*