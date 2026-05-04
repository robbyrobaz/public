# Kalshi Edge — Autonomous Prediction-Market Trading System

A fully autonomous trading system that scans [Kalshi](https://kalshi.com) every 15 minutes, places paper bets across ~65 strategies, and flips proven strategies to live real-money trading with strict risk controls.

**Status (May 4, 2026):** Live sleeve cut to **one workhorse**. `crypto_15m_momentum_v2` running with **+$127.19 net / 40% WR / PF 2.01 over n=231 trades**. `crypto_daily_fade_v2` on watch (n=11). Two strategies (`favorite_no_aggressive_v2` and `gemini_picks_v2`) demoted today after live results diverged from paper. **All AI-powered pickers disabled** — they were paper-positive but live-negative once API costs landed.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  systemd timer (every 15 min) — kalshi-paper-v2.timer             │
│                                                                    │
│  ┌──────────────┐   ┌────────────────────────────────────────┐    │
│  │  resolver    │ → │  ~65 mechanical scanners (no AI)       │    │
│  │  (Kalshi API)│   │  crypto / sports / weather / arb       │    │
│  └──────────────┘   └────────────────────────────────────────┘    │
│           ↓                              ↓                         │
│           └──────────────────────────────┘                         │
│                              ↓                                     │
│                     paper_v2_trades.jsonl                          │
│                              ↓                                     │
│                        promotion_check                             │
│                  (gates n≥20, PF, WR, Sharpe, DD)                  │
│                              ↓                                     │
│                  data/live_sleeve_eligible.json                    │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│  Per-strategy live runners (every 15 min when promoted)            │
│   • Place real orders via Kalshi /portfolio/orders                 │
│   • Confirm fills via /portfolio/orders/{id} polling               │
│   • Re-read book before placement (concurrent-write dedup)         │
│   • small_balance_guard for per-bet correctness                    │
│   • Refuse to fire if status != 'ready'                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## Mechanical-Only Strategy Library (~65 strategies, zero AI)

Pure price-filter rules running every 15 min:
- **Crypto momentum / fade / breakout / reversion** — 15-min and hourly KXBTC/KXETH ladders
- **Sports favorite NO** — high-OI NBA/NHL/MLB/NCAA underdog hedges (4 new MLB/NHL strategies added May 4)
- **MLB player props** — strikeout NO, hits-allowed YES at 5–12¢, total-bases NO
- **NHL playoff specials** — totals under, dog covers, underdog moneylines
- **Cross-platform arb** — Kalshi vs Polymarket title-matched divergence
- **Mention markets** — Trump/CEO/politician statement scanners
- **Weather / macro events** — KXHIGH*/KXCPI/KXFOMC scanners (middle-band only, no extreme tails)
- **Multi-outcome overround fade** — 3+ way exclusive events with sum-yes overround in [5–30%]
- **Crypto ladder lognormal model** — Black-Scholes-style probability modeling using realized volatility from CoinGecko + 1-min Blofin candles
- **Data-enriched scanners** — pull live NQ futures (1-min IBKR feed) and CoinGecko 5-min candles to inform crypto + macro bets with grounded numeric context

### What was tried and shut off (May 4)

The system previously included an **AI Picker Showdown**: 4-5 frontier LLMs (Grok, Claude, Gemini, DeepSeek, Qwen) hitting the same candidate batch and competing on paper PnL + Brier score. After 1-2 weeks of live data:

- `gemini_picks_v2`: paper PF 4.76 → **live PF 0.78 over n=14** (12 losses)
- `data_enriched_grok_picks_v2`: ~$0.05 cost per pick × 30+ candidates per cycle = **~$1.50/run overhead** that paper never modeled

**Decision:** disabled the gemini live timer, demoted the strategy in `live_sleeve_eligible.json`, and commented `data_enriched_grok_picker.py` out of the paper runner. Total live AI spend now: $0/day. The system is mechanical-only until/unless an AI strategy can demonstrate edge survival under realistic per-call cost overhead.

---

## Risk Philosophy

| | Paper (data factory) | Live (real money) |
|---|---|---|
| Position cap | None | 3 concurrent |
| Dollar exposure cap | None | $5–$25/bet (per-strategy) |
| Daily loss halt | None | none — strategy-level kill instead |
| **Strategy-level kill** | n/a | **PF<0.70 on n≥30 OR WR<25% on n≥25 OR ≤−20% sleeve drawdown** |
| Per-bet correctness | None | price band $0.10-$0.50, contracts 5-50, per-event dedup, fresh-read dedup before order |

**Paper is unconstrained** — it's the data factory. Every loss is signal. Every strategy gets to test itself.

**Live uses a single kill switch** with a sticky `live_killed` flag that prevents auto-re-promotion on stale paper data — protects against the "paper looks great but live keeps losing" trap.

---

## Order Confirmation (Defense-in-Depth)

Every live order goes through TWO authoritative checks:

1. **At placement** — `place_order` returns immediately, then `confirm_fill(order_id, timeout=60s)` polls `/portfolio/orders/{id}` until status is `executed`/`canceled`/`expired`. Records actual `fill_count_fp`, `maker_fill_cost_dollars`, `taker_fees_dollars` (not intended values).

2. **At settlement** — `resolve_live_settled` reads `/portfolio/settlements` directly. Uses Kalshi's authoritative `revenue / cost / fees` for PnL. Does NOT recompute from our records (partial fills break that math).

This caught a bug early: 2 of our first 17 "trades" had been recorded but never actually filled — Kalshi confirmed `revenue: $0, cost: $0`. They were inflating reported PnL by $6.35. Defense-in-depth wins over silent corruption.

A duplicate-bet bug found May 3 added a third defense layer: re-read the book immediately before each order placement. Concurrent runner writes were overwriting `live_sleeve_trades.jsonl` while another runner's resolver-save was in flight, occasionally hiding existing positions from the dedup check.

---

## Single-Page Dashboard

Mobile-first SPA at `omen-claw.tail76e7df.ts.net:8898`:

- **Live tab** — sleeve balance, today PnL, **per-strategy live performance cards** (PnL/WR/PF tiles + drawdown bar), $/Day and ROI%/Day tiles (deposit-independent, calculated as daily return on average deployed capital), **cumulative PnL chart with per-strategy lines on 4-hour buckets**, ROI% per 4h bucket bar chart
- **Paper tab** — total PnL, per-strategy bar chart, win-rate, profit factor as primary metric
- **Promotion tab** — per-strategy gate-check chips, LIVE status badges (🟢 live / 🛑 live-killed / ⏸ demoted), $100 sleeve simulator with realistic-fill / slip-aware simulation
- **Risk tab** — per-bet correctness rules + strategy-level kill thresholds + open exposure capacity bars

Single shell across all tabs (no jarring transitions). Auto-refresh every 30s, pauses when tab hidden.

---

## Tech Stack

| Layer | Tech |
|---|---|
| Schedulers | systemd user timers (every 15 min) |
| Strategy scanners | Python (one file per strategy, ~50 lines each, shared `_strategy_template_v2`) |
| Cross-platform matcher | Polymarket gamma API + custom title-matching with category whitelist |
| Storage | JSONL append-only logs (`paper_v2_trades.jsonl`, `live_sleeve_trades.jsonl`) |
| Dashboard | Flask + vanilla JS + Chart.js (single-page app, no build step) |
| Local data | NQ 1-min CSV (live IBKR feed), CoinGecko 5-min candles, Blofin 1-min parquet |

Total cost: **~$0/day**. Kalshi API is free. AI calls are off.

---

## Selected Lessons (the hard-earned ones)

1. **Sports markets show paper-to-live regime collapse, crypto markets do not.** Three strategies in a row (`nba_spread_yes`, `gemini_picks_v2`, `favorite_no_aggressive_v2`) all dropped 15-30 percentage points of WR going from paper to live. Live execution lands outside the paper-modeled fill curve because sports books move the moment a NO bet hits the favorite leg. Crypto 15m markets have $0.01 spreads and millisecond-level depth — the paper edge survives. Going forward: sports promotion requires `live_paper_sim` (realistic fill + slip) edge survival, not raw paper edge.

2. **AI pickers were paper-positive and live-negative.** Gemini's paper PF was 4.76; live was 0.78 over 14 trades. The per-call cost overhead (~$1.50/run) plus the same execution gap that hits manual sports strategies eats the edge. Disabling LLM strategies until the simulated-realistic-fill paper book includes API spend.

3. **Day-of-week matters.** Crypto 15m has a **−29.6% ROI Sunday effect** (n=30) and **+35.1% Saturday effect** (n=28). Likely thin Asian-session orderbooks early Sunday. A bad-feeling crypto streak should be checked against day-of-week before blamed on the strategy.

4. **The candlestick endpoint introduces look-ahead bias.** A backtest that "found" cheap-YES sports props at 30%+ EV went 0/19 live — root cause was the candlestick endpoint returning post-close prices. Killed an entire strategy family. Always use the markets endpoint for true pre-event prices.

5. **Limit orders fill seconds-to-minutes after submission.** Recording intended size + price as if filled produces phantom PnL. Always poll `/portfolio/orders/{id}` after placement.

6. **Concurrent runner writes can race the deduper.** Multiple per-strategy runners share `live_sleeve_trades.jsonl`. One runner's `resolve_live_settled.save()` overwriting the file while another is appending can briefly hide entries from the start-of-run book read. Solution: re-read the book right before placing each order as a fresh-dedup check.

7. **The kill switch was tuned too tight.** Killing at PF<1.0 on n=10 has ~35% false-positive rate against TRUE PF=1.5 strategies. Loosened to PF<0.70 on n=30 — the −20% sleeve drawdown is the actual capital guard, PF/WR need bigger samples to be trustworthy.

8. **Don't parse ticker names for time data.** Kalshi tickers like `KXBTC15M-26APR281245-45` use Eastern Time, not UTC. Pulling `close_time` from `/markets/{ticker}` is the only reliable source. (NBA quirk: `close_time` is the playoff series end while `expected_expiration_time` is the actual game end — use the earlier of the two.)

9. **n=20 is the right call-it-broken threshold for sports strategies.** With WR 15% on n=20 vs expected 50%, the binomial probability of being legitimately at 50% is ~0.001 — clearly broken, not variance. Killing at n=14 (gemini) was overdue; killing at n=20 (favorite_no_aggressive) was right.

---

*Built by [@robbyrobaz](https://github.com/robbyrobaz). Source code is private; this page is a public summary.*
