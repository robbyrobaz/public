# Kalshi Edge — Autonomous Prediction-Market Trading System

Autonomous trading system that scans [Kalshi](https://kalshi.com) every 15 minutes, runs ~60 paper strategies, and surfaces winners that an operator manually promotes to a real-money live sleeve.

**Status (May 12, 2026):** Two live strategies, both crypto 15-minute, both mechanical (no AI):

| Strategy | Sleeve | Live record |
|---|---|---|
| `crypto_15m_momentum_v2` | $200 | n=516 · WR 36% · PF 1.05 · **+$143** |
| `crypto_15m_mean_reversion_v2` | $100 | just promoted on n=716 paper · WR 23.7% · PF 1.19 · z=1.86 |

All AI strategies disabled. Auto-promotion/demotion disabled. **Total live spend: $0/day.**

---

## The pattern that's actually working

Every strategy that's profitable in live shares the same signature:

| Pattern | Why |
|---|---|
| **Crypto 15-minute markets only** | $0.01 spreads, deep millisecond-level books. Sports/daily/AI strategies have all broken in paper-to-live transition. |
| **NO-side bets concentrated** | NO carries the strategy. Mean reversion: NO +13.9% ROI / YES +4.9%. BTC NO at $0.15-$0.25 is the all-time MVP bucket. |
| **BTC > ETH** | Both strategies show BTC ROI ~3× ETH ROI per dollar deployed. Deeper book, tighter spreads. |
| **Asymmetric R:R, low WR is fine** | Wins are 2-3× the size of losses. WR can be 23-40% and still net profitable. Don't promote on WR — promote on PF + sample size. |
| **Bid at displayed ask, never +1¢** | The single biggest live-execution lesson. Forcing fills above ask is adverse selection. |
| **Trend filter** | Skip when underlying just moved >1% in last 60 min. Both strategies are fade-style; trends kill fades. |

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  systemd timer (every 15 min) — kalshi-paper-v2.timer             │
│                                                                    │
│  ┌──────────────┐   ┌────────────────────────────────────────┐    │
│  │  resolver    │ → │  ~60 mechanical scanners (no AI)       │    │
│  │  (Kalshi API)│   │  crypto / sports / weather / arb       │    │
│  └──────────────┘   └────────────────────────────────────────┘    │
│           ↓                              ↓                         │
│           └──────────────────────────────┘                         │
│                              ↓                                     │
│                     paper_v2_trades.jsonl                          │
│                              ↓                                     │
│                  promotion_check (read-only HTML)                  │
│                              ↓                                     │
│              data/live_sleeve_eligible.json                        │
│                  (operator-edited only)                            │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│  Per-strategy live runners (every 15 min when status=ready)        │
│   • Place real orders at displayed no_ask (NOT ask+1)              │
│   • Trend filter: skip if |60min move| > 1.0%                      │
│   • confirm_fill polling (60s)                                     │
│   • Fresh book re-read before placement                            │
│   • Refuse to fire if status != 'ready' in eligible JSON           │
└────────────────────────────────────────────────────────────────────┘
```

---

## Hard-earned lessons

### 1. The `+1¢` bug — adverse-selection trap

A May-2 change had the live runner bid `ask + 1 cent` to "prevent resting-order no-fills." It worked: no-fills dropped from 7 → 2. **And live WR collapsed from 40% → 21% over 47 trades**, costing ~$200.

Bidding above ask forces fills on orders that would have rested unfilled when the book moved against us. Those forced fills are adversely selected — the book moved for a reason. After revert: WR snapped back to 36-39% within 36 hours.

**Rule:** never pay above the displayed ask. Limit orders that don't fill are the strategy correctly skipping moved-against-us trades.

### 2. Paper-to-live regime collapse

Five strategies have shown this pattern:
1. `nba_spread_yes` — sports paper 25-33% → live 10%
2. `gemini_picks_v2` — AI picker paper PF 4.76 → live 0.78
3. `favorite_no_aggressive_v2` — sports paper PF 2.69 → live 0.99
4. `data_enriched_grok_picks_v2` — same pattern
5. **`crypto_daily_fade_v2`** — paper PF 1.92 → live 0.16 in trending BTC (May 12)

The first four are sports/AI. The fifth is crypto but **daily timeframe** — and daily fades only get 1 chance per day to recover from a trending regime. 15-minute fades survive trends because they rebalance 96× per day.

**Rule:** sports promotion requires `live_paper_sim` (realistic fill + slip) edge survival. AI strategies need to model API cost. Daily strategies need trend-regime testing.

### 3. Auto-killers kill mid-investigation

When the +1¢ bug was bleeding, the auto-promoter correctly fired at -20% sleeve drawdown. The operator wanted to investigate the bug first; the auto-promoter had already demoted. Disabled May 5. Promotion is now manual-only via `live_sleeve_eligible.json`. Two-layer protection in place.

### 4. Day-of-week matters

Crypto 15m has a real **-29.6% ROI Sunday effect** (n=30) and **+35.1% Saturday effect** (n=28). Likely thin Asian-session orderbooks early Sunday. A bad-feeling crypto streak should be checked against day-of-week before blamed on the strategy.

### 5. Trending-tape kills momentum-fade

Live data: chop days (0 hours with >1% BTC move) ROI **+14.5%** / n=186; trending days (≥2 hours >1%) ROI **-25%** / n=21. The strategy buys NO at YES_ASK 0.55-0.80 — wins when 15m strikes don't get hit (chop), loses when crypto trends because strikes keep clearing.

**Fix:** 60-min `recent_pct_move` filter from Coinbase public spot. Skip if |move| > 1.0%. Fail-open on network errors.

### 6. Concurrent runner writes can race the deduper

Multiple per-strategy runners share `live_sleeve_trades.jsonl`. One runner's `resolve_live_settled.save()` overwriting the file while another is appending can briefly hide entries from the start-of-run book read. Caused 3 duplicate NBA bets May 2-3. Fix: re-read the book right before placing each order.

### 7. Limit orders fill seconds-to-minutes after submission

Recording intended size + price as if filled produces phantom PnL. 2 of the first 17 "trades" had been recorded but never actually filled — Kalshi confirmed `revenue: $0, cost: $0`. Always poll `/portfolio/orders/{id}` after placement.

### 8. The candlestick endpoint introduces look-ahead bias

A V1 backtest "found" cheap-YES sports props at 30%+ EV that went 0/19 live. Root cause: candlestick endpoint returns post-close prices. Killed an entire strategy family. Always use the markets endpoint for true pre-event prices.

### 9. Asymmetric R:R is the whole game

Wins average $25-30, losses average $9-18 across both live strategies. WR 23-40% is profitable because R:R is 2-3:1. Don't promote based on WR; promote based on PF + sample size. Low-WR strategies feel terrifying for the first 20-30 live trades — be psychologically ready for that.

### 10. Cap visualizations, never the math

A 4-hour bucket showed 1307% ROI on the dashboard because 5 cheap-NO bets ($5.35 total cost) all settled as wins on the same NBA game ($69.91 payout). Math is correct; chart unreadable. Fix: cap bar at ±200%, show un-clipped truth in tooltip.

### 11. Don't parse ticker names for time data

Kalshi tickers like `KXBTC15M-26APR281245-45` use Eastern Time, not UTC. Pulling `close_time` from `/markets/{ticker}` is the only reliable source.

### 12. Manual operator override > algorithmic gates

The full sequence of failures + recoveries took 11 days. At every decision point, the operator caught nuance the auto-system couldn't:
- "Don't demote it!" when auto-killer fired during bug investigation
- "Check the code before changing it" when initial diagnosis was wrong
- "We have enough data" when caution would have delayed promotion

Trust the human in the loop. Build instruments, not autopilot.

---

## Tech stack

| Layer | Tech |
|---|---|
| Schedulers | systemd user timers (every 15 min) |
| Strategy scanners | Python — one file per strategy, ~50 lines each |
| Cross-platform matcher | Polymarket gamma API + custom title-matching |
| Storage | JSONL append-only logs |
| Dashboard | Flask + vanilla JS + Chart.js (no build step) |
| Price feeds | Coinbase public spot (trend filter), CoinGecko 5m, Blofin 1m parquet |

Total live spend: **~$0/day**. Kalshi API is free. AI calls are disabled.

---

*Built by [@robbyrobaz](https://github.com/robbyrobaz). Source code is private; this page is a public summary.*
