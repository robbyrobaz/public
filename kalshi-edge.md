# Kalshi Edge — Autonomous Prediction-Market Trading System

A fully autonomous trading system that scans [Kalshi](https://kalshi.com) every 15 minutes, places paper bets across 40+ strategies, and flips proven strategies to live real-money trading with strict risk controls.

**Status (Apr 27, 2026):** Paper book has 24-day track record across 40+ strategies. First live sleeve (`crypto_15m_momentum_v2`) was promoted then auto-killed after 5-loss streak — system protected capital exactly as designed.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  systemd timer (every 15 min) — kalshi-paper-v2.timer             │
│                                                                    │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────────┐     │
│  │  resolver    │ → │  41 mechanical │ → │  AI pickers      │     │
│  │  (Kalshi API)│   │  scanners      │   │  (6 LLM models)  │     │
│  └──────────────┘   └────────────────┘   └──────────────────┘     │
│           ↓                  ↓                   ↓                 │
│           └──────────────────┴───────────────────┘                 │
│                              ↓                                     │
│                     paper_v2_trades.jsonl                          │
│                              ↓                                     │
│              promotion_check + live_sleeve_promoter                │
│                              ↓                                     │
│                  data/live_sleeve_eligible.json                    │
└────────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────────┐
│  Per-strategy live runners (every 15 min when promoted)            │
│   • Place real orders via Kalshi /portfolio/orders                 │
│   • Confirm fills via /portfolio/orders/{id} polling               │
│   • small_balance_guard for per-bet correctness                    │
│   • Refuse to fire if status != 'ready'                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## Two-Brain Architecture

### Brain 1 — Mechanical Scanners (40+ strategies, no AI)
Pure price-filter rules running every 15 min:
- **Crypto momentum / fade / breakout / reversion** — 15-min and hourly KXBTC/KXETH ladders
- **Sports favorite NO bets** — high-OI NBA/NHL/MLB/NCAA underdog hedges
- **Cross-platform arb** — Kalshi vs Polymarket title-matched divergence
- **Mention markets** — Trump/CEO/politician statement scanners
- **Weather / macro events** — KXHIGH*/KXCPI/KXFOMC scanners
- **Crypto ladder lognormal model** — Black-Scholes-style probability modeling using realized volatility from 1-min Blofin candles

### Brain 2 — AI Picker Showdown (6 LLMs on same candidate batch)

A unified picker fetches ~40-80 live Kalshi candidates and sends the same JSON array to multiple frontier models in parallel. Each model returns:
1. `predictions[]` — per-candidate probability scores (for calibration analysis)
2. `picks[]` — top-N high-conviction bets (for paper-betting)

| Model | Path | Edge |
|---|---|---|
| **Grok 4.1 Fast** | OpenRouter (web + X search) | Real-time injury news, breaking tweets |
| **Claude Opus 4.7** | Local CLI (free) | Reasoning, calibration honesty |
| **Gemini 2.5 Pro** | OpenRouter (reasoning) | Long-context probability judgment |
| **DeepSeek V3.2** | OpenRouter (web) | Cheap reasoning, often surprisingly strong |
| **Qwen 3 235B** | OpenRouter (web) | Relative ranking |

Plus a **specialized Grok-curated picker** running 6x/day on a pre-filtered pool of news-driven markets (sports w/in 6h, mention markets, daily crypto, macro events) with Polymarket cross-platform price enrichment.

Each model's picks land as a separate paper strategy (`grok_picks_v2`, `claude_picks_v2`, etc.) so PnL/PF/WR/Brier-score accumulate independently — bad models self-eliminate via promotion gates.

---

## Risk Philosophy

| | Paper (data factory) | Live (real money) |
|---|---|---|
| Position cap | None | 3 concurrent |
| Per-strategy cap | None | n/a |
| Dollar exposure cap | None | $5/bet |
| Daily loss halt | None | none — strategy-level kill instead |
| **Strategy-level kill** | n/a | **PF<1.0 on n≥10 OR WR<30% on n≥20 OR ≤−20% sleeve drawdown** |
| Per-bet correctness | None | price band $0.10-$0.50, contracts 5-10, max bet $5, per-event dedup |

**Paper is unconstrained** — it's the data factory. Every loss is signal. Every strategy gets to test itself. Promotion gates filter out losers before they ever reach live.

**Live uses a single kill switch**: the promoter evaluates rolling LIVE performance and demotes when the strategy actually fails. No arbitrary day-boundary halts, no profit locks. If the strategy works it works; if it fails it stops.

After demotion, a sticky `live_killed` flag prevents auto-re-promotion on stale paper data. Manual `--reset` required to retry — protects against the "paper looks great but live keeps losing" trap.

---

## Order Confirmation (Defense-in-Depth)

Every live order goes through TWO authoritative checks:

1. **At placement** — `place_order` returns immediately, then `confirm_fill(order_id, timeout=15s)` polls `/portfolio/orders/{id}` until status is `executed`/`canceled`/`expired`. Records actual `fill_count_fp`, `maker_fill_cost_dollars`, `taker_fees_dollars` (not intended values).

2. **At settlement** — `resolve_live_settled` reads `/portfolio/settlements` directly. Uses Kalshi's authoritative `revenue / cost / fees` for PnL. Does NOT recompute from our records (partial fills break that math).

This caught a critical bug early: 2 of our first 17 "trades" had been recorded but never actually filled — Kalshi confirmed `revenue: $0, cost: $0`. They were inflating our reported PnL by $6.35 and would have continued doing so. Defense-in-depth wins over silent corruption.

---

## Single-Page Dashboard

Mobile-first SPA at `omen-claw.tail76e7df.ts.net:8898`:

- **Live tab** — sleeve balance (reconciles exactly to Kalshi cash), today PnL, open positions sorted by closes-in time, live runner candidates, recent trades
- **Paper tab** — total PnL, per-strategy bar chart, win-rate, profit factor as primary metric
- **AI Showdown tab** — 6-model leaderboard with PnL + Brier score + directional accuracy + confidence distribution per model
- **Promotion tab** — per-strategy gate-check chips, LIVE status badges (🟢 live / 🛑 live-killed / ⏸ demoted), $100 sleeve simulator
- **Risk tab** — per-bet correctness rules + strategy-level kill thresholds + open exposure capacity bars

Single shell across all tabs (no jarring transitions). Auto-refresh every 30s, pauses when tab hidden.

---

## Tech Stack

| Layer | Tech |
|---|---|
| Schedulers | systemd user timers (every 15 min) |
| Strategy scanners | Python (one file per strategy, ~50 lines each) |
| AI calls | OpenRouter (`x-ai/grok-4.1-fast`, `google/gemini-2.5-pro`, `deepseek/deepseek-v3.2`, `qwen/qwen3-vl-235b-a22b-instruct`) + Claude CLI |
| Cross-platform matcher | Polymarket gamma API + custom title-matching with category whitelist |
| Storage | JSONL append-only logs (`paper_v2_trades.jsonl`, `live_sleeve_trades.jsonl`, `ai_predictions.jsonl`) |
| Dashboard | Flask + vanilla JS + Chart.js (single-page app, no build step) |
| Crypto vol | Blofin 1-min candle parquet files for realized-vol calculations |

Total cost: **~$2.20/day** for AI calls (Grok curated 6x/day + 5-model showdown 2x/day). Kalshi API is free.

---

## Selected Lessons (the hard-earned ones)

1. **The candlestick endpoint introduces look-ahead bias.** A backtest that "found" cheap-YES sports props at 30%+ EV went 0/19 live — root cause was the candlestick endpoint returning post-close prices. Killed an entire strategy family. Always use the markets endpoint for true pre-event prices.

2. **Limit orders fill seconds-to-minutes after submission.** Recording intended size + price as if filled produces phantom PnL. Always poll `/portfolio/orders/{id}` after placement.

3. **Paper auto-pauses → paper data freezes → promoter must evaluate LIVE data.** A bug where the promoter detected live failure but the same run re-promoted on stale paper data cost real money. Sticky `live_killed` flag fixed it.

4. **AI consensus ≠ AI being right.** When 4 of 5 models bet the same direction (Sixers Game 4, Lakers Game 4), they were just reading the same conventional sports narrative — both lost. Independent agreement (Grok + Gemini independently picking the same Royals MLB win) is more meaningful signal.

5. **Asking AI for "top picks only" inflates confidence.** The same model went from 0.78+ confidences (when asked for top picks) to honest 0.40-0.65 distribution (when asked to score every candidate). Ask the AI to evaluate everything — extract picks downstream.

---

*Built by [@robbyrobaz](https://github.com/robbyrobaz). Source code is private; this page is a public summary.*
