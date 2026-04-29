# Kalshi Edge — Autonomous Prediction-Market Trading System

A fully autonomous trading system that scans [Kalshi](https://kalshi.com) every 15 minutes, places paper bets across 65 strategies, and flips proven strategies to live real-money trading with strict risk controls.

**Status (Apr 28, 2026):** Two live sleeves active — `crypto_15m_momentum_v2` (re-promoted after paper recovered to PF 1.43, currently 2W/2L net +$2.62 in fresh window) and `favorite_no_aggressive_v2` (operator-promoted at n=10). The first live attempt was auto-killed correctly after a 5-loss streak — system protected capital, paper-side recovered, re-promotion proved viable.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│  systemd timer (every 15 min) — kalshi-paper-v2.timer             │
│                                                                    │
│  ┌──────────────┐   ┌────────────────┐   ┌──────────────────┐     │
│  │  resolver    │ → │  ~55 mechanical│ → │  AI pickers      │     │
│  │  (Kalshi API)│   │  scanners      │   │  (4 LLM models)  │     │
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

### Brain 1 — Mechanical Scanners (~55 strategies, no AI)
Pure price-filter rules running every 15 min:
- **Crypto momentum / fade / breakout / reversion** — 15-min and hourly KXBTC/KXETH ladders
- **Sports favorite NO bets** — high-OI NBA/NHL/MLB/NCAA underdog hedges
- **MLB player props** — strikeout NO, hits-allowed YES at 5–12¢, total-bases NO
- **NHL playoff specials** — totals under, dog covers, underdog moneylines
- **Cross-platform arb** — Kalshi vs Polymarket title-matched divergence
- **Mention markets** — Trump/CEO/politician statement scanners
- **Weather / macro events** — KXHIGH*/KXCPI/KXFOMC scanners (middle-band only, no extreme tails)
- **Multi-outcome overround fade** — 3+ way exclusive events with sum-yes overround in [5–30%]
- **Crypto ladder lognormal model** — Black-Scholes-style probability modeling using realized volatility from CoinGecko + 1-min Blofin candles
- **Data-enriched scanners** — pull live NQ futures (1-min IBKR feed) and CoinGecko 5-min candles to inform crypto + macro bets with grounded numeric context

### Brain 2 — AI Picker Showdown (4 LLMs on same candidate batch)

A unified picker fetches ~40-80 live Kalshi candidates and sends the same JSON array to multiple frontier models in parallel. Each model returns:
1. `predictions[]` — per-candidate probability scores (for calibration analysis)
2. `picks[]` — top-N high-conviction bets (for paper-betting)

| Model | Path | Edge |
|---|---|---|
| **Grok 4.1 Fast** | OpenRouter (web + X search) | Real-time injury news, breaking tweets |
| **Claude Opus 4.7** | Local CLI (free) | Reasoning, calibration honesty |
| **Gemini 2.5 Pro** | OpenRouter (reasoning) | Long-context probability judgment |
| **DeepSeek V3.2** | OpenRouter (web) | Cheap reasoning, often surprisingly strong |

*(Qwen 3 VL was dropped Apr 28 — the vision-language model returned valid JSON with empty arrays because OpenRouter's web plugin isn't routed through VL models, and it refused to score from priors alone.)*

Plus two specialized pickers on a 4-hour cadence:
- **Curated picker** — Grok and Gemini in parallel (logged as separate strategies) on a pre-filtered news-driven pool: sports within 6h, mention markets, daily crypto, macro events. Polymarket-enriched.
- **Data-enriched Grok picker** — same model, but the prompt prepends a JSON header with live local data (NQ futures % moves, BTC/ETH/SOL spot + recent change + 24h realized volatility). Tests whether grounded numeric context beats web-search-only.

Each model + each variant lands as a separate paper strategy (`grok_picks_v2`, `grok_curated_picks_v2`, `gemini_curated_picks_v2`, `data_enriched_grok_picks_v2`, etc.) so PnL/PF/WR/Brier-score accumulate independently — bad models self-eliminate via promotion gates.

---

## Risk Philosophy

| | Paper (data factory) | Live (real money) |
|---|---|---|
| Position cap | None | 3 concurrent |
| Per-strategy cap | None | n/a |
| Dollar exposure cap | None | $5/bet |
| Daily loss halt | None | none — strategy-level kill instead |
| **Strategy-level kill** | n/a | **PF<0.70 on n≥30 OR WR<25% on n≥25 OR ≤−20% sleeve drawdown** |
| Per-bet correctness | None | price band $0.10-$0.50, contracts 5-10, max bet $5, per-event dedup |

**Paper is unconstrained** — it's the data factory. Every loss is signal. Every strategy gets to test itself. Promotion gates filter out losers before they ever reach live.

**Live uses a single kill switch**: the promoter evaluates rolling LIVE performance and demotes when the strategy actually fails. No arbitrary day-boundary halts, no profit locks. If the strategy works it works; if it fails it stops.

After demotion, a sticky `live_killed` flag prevents auto-re-promotion on stale paper data. Manual reset required to retry — protects against the "paper looks great but live keeps losing" trap.

The kill thresholds were **loosened from PF<1.0/n=10 → PF<0.70/n=30** in late April after a re-promoted strategy proved the original n=10 gate fired during early-sample variance: a TRUE PF=1.5 strategy has a ~35% chance of randomly seeing PF<1.0 in its first 10 trades. The −20% sleeve drawdown remains the actual capital guard; PF/WR are judgment gates that need bigger samples.

---

## Order Confirmation (Defense-in-Depth)

Every live order goes through TWO authoritative checks:

1. **At placement** — `place_order` returns immediately, then `confirm_fill(order_id, timeout=15s)` polls `/portfolio/orders/{id}` until status is `executed`/`canceled`/`expired`. Records actual `fill_count_fp`, `maker_fill_cost_dollars`, `taker_fees_dollars` (not intended values).

2. **At settlement** — `resolve_live_settled` reads `/portfolio/settlements` directly. Uses Kalshi's authoritative `revenue / cost / fees` for PnL. Does NOT recompute from our records (partial fills break that math).

This caught a critical bug early: 2 of our first 17 "trades" had been recorded but never actually filled — Kalshi confirmed `revenue: $0, cost: $0`. They were inflating our reported PnL by $6.35 and would have continued doing so. Defense-in-depth wins over silent corruption.

---

## Single-Page Dashboard

Mobile-first SPA at `omen-claw.tail76e7df.ts.net:8898`:

- **Live tab** — sleeve balance, today PnL, **per-strategy live performance cards** (live PnL/WR/PF tiles + drawdown bar + trades-until-kill-check progress + last-result pill), **cumulative PnL chart with per-strategy lines on 4-hour buckets** (TOTAL bold + thin colored line per strategy), recent trades with proper UTC close times pulled from Kalshi API
- **Paper tab** — total PnL, per-strategy bar chart, win-rate, profit factor as primary metric
- **AI Showdown tab** — model leaderboard with PnL + Brier score + directional accuracy + confidence distribution + **per-strategy `$/call`** (so 3 strategies sharing the Grok model each show their own real cost), **Net PnL by Model bar chart** (gross PnL vs API cost drag), **Spend by Model panel** (OpenRouter daily spend stacked-bar)
- **Promotion tab** — per-strategy gate-check chips, LIVE status badges (🟢 live / 🛑 live-killed / ⏸ demoted), $100 sleeve simulator
- **Risk tab** — per-bet correctness rules + strategy-level kill thresholds + open exposure capacity bars

Single shell across all tabs (no jarring transitions). Auto-refresh every 30s, pauses when tab hidden.

---

## Tech Stack

| Layer | Tech |
|---|---|
| Schedulers | systemd user timers (every 15 min) |
| Strategy scanners | Python (one file per strategy, ~50 lines each) |
| AI calls | OpenRouter (`x-ai/grok-4.1-fast`, `google/gemini-2.5-pro`, `deepseek/deepseek-v3.2`) + Claude CLI |
| Cross-platform matcher | Polymarket gamma API + custom title-matching with category whitelist |
| Storage | JSONL append-only logs (`paper_v2_trades.jsonl`, `live_sleeve_trades.jsonl`, `ai_predictions.jsonl`, `llm_call_log.jsonl`) |
| Dashboard | Flask + vanilla JS + Chart.js (single-page app, no build step) |
| Local data | NQ 1-min CSV (live IBKR feed), CoinGecko 5-min candles, Blofin 1-min parquet |

Total cost: **~$0.75–$2/day** for AI calls. Tracked per-call via `llm_call_log.jsonl` rather than the OpenRouter dashboard total — shared API keys roll up calls from many systems, so attributing OR's total spend to Kalshi inflated costs ~7×. Kalshi API is free.

---

## Selected Lessons (the hard-earned ones)

1. **The candlestick endpoint introduces look-ahead bias.** A backtest that "found" cheap-YES sports props at 30%+ EV went 0/19 live — root cause was the candlestick endpoint returning post-close prices. Killed an entire strategy family. Always use the markets endpoint for true pre-event prices.

2. **Limit orders fill seconds-to-minutes after submission.** Recording intended size + price as if filled produces phantom PnL. Always poll `/portfolio/orders/{id}` after placement.

3. **Paper auto-pauses → paper data freezes → promoter must evaluate LIVE data.** A bug where the promoter detected live failure but the same run re-promoted on stale paper data cost real money. Sticky `live_killed` flag fixed it.

4. **AI consensus ≠ AI being right.** When 4 of 5 models bet the same direction (Sixers Game 4, Lakers Game 4), they were just reading the same conventional sports narrative — both lost. Independent agreement (Grok + Gemini independently picking the same Royals MLB win) is more meaningful signal.

5. **Asking AI for "top picks only" inflates confidence.** The same model went from 0.78+ confidences (when asked for top picks) to honest 0.40-0.65 distribution (when asked to score every candidate). Ask the AI to evaluate everything — extract picks downstream.

6. **The kill switch was tuned too tight.** Killing at PF<1.0 on n=10 has ~35% false-positive rate against TRUE PF=1.5 strategies. Loosened to PF<0.70 on n=30 — the −20% sleeve drawdown is the actual capital guard, PF/WR need bigger samples to be trustworthy.

7. **Shared API keys hide true cost.** Pulling OpenRouter total spend and attributing it to Kalshi inflated costs ~7×. Per-call token logging (Kalshi-only) revealed real spend is ~$0.75/week. And vision-language models don't get web plugins routed — Qwen 3 VL silently returned empty arrays for that reason.

8. **Don't parse ticker names for time data.** Kalshi tickers like `KXBTC15M-26APR281245-45` use Eastern Time, not UTC. Pulling `close_time` from `/markets/{ticker}` is the only reliable source. (NBA quirk: `close_time` is the playoff series end while `expected_expiration_time` is the actual game end — use the earlier of the two.)

---

*Built by [@robbyrobaz](https://github.com/robbyrobaz). Source code is private; this page is a public summary.*
