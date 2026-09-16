# Second-Pass Review — Intraday Shadow-Mode Trading System v2

**Reviewer:** Senior quant / systems architect, second pass  
**Date:** 2026-09-15  

---

## 1. Flaws and Blind Spots That Survived the Rewrite

### 1.1 The IEX problem is identified but not resolved

§3.1 correctly names the IEX-only data problem and offers two options. But the damage is worse than stated. Signal families B (ORB volume confirmation), C (VWAP — which is *volume-weighted*), and D (volume-spike continuation) all depend on volume being representative. On IEX's ~2-3% market share, you're not measuring "volume" with noise — you're measuring a biased sample whose bias varies by ticker, time-of-day, and volatility regime. Option (2) — "trade off IEX, record consolidated on a 15-minute delay" — means every shadow-mode *decision* is made on data you've already acknowledged is distorted, and you only learn this 15 minutes too late to matter.

The honest resolution: if you're paying the $29/month for Polygon, use Polygon's real-time websocket for the 15-20 tickers that pass your 09:28 filter. You don't need full-market real-time consolidated data. You need consolidated data for the handful of names you're actually considering trading, and Polygon Starter gives you 5 websocket connections. That's enough for one-at-a-time trading. If the tier doesn't support websockets, poll the snapshot endpoint every 15 seconds for your shortlist — you're already on a 15-second loop. Either way, stop pretending IEX volume is a usable input for VWAP or volume-spike detection. It isn't.

### 1.2 Entry order type is completely unspecified

The doc specifies stop placement, position sizing, exit mechanics, and time barriers — but never says what *entry orders* look like. Market order? Limit at the ask? Limit at mid with a short timeout? This matters enormously:

- For ORB breakouts (signal B), you need immediacy — a limit order 2 cents above the breakout bar's high will miss fast moves and only fill on the ones that fail. That's adverse selection, and it's a real cost that the backtest's "entry at next bar's open" doesn't capture.
- For VWAP reversion (signal C), you can afford patience — a limit at VWAP + offset with a 2-minute timeout is reasonable and will get better fills than a market order.
- For volume-spike continuation (signal D), the edge decays in 5-15 minutes. A patient limit order is a contradiction.

The fill simulator (§7.1) models "entry at next bar's open" for the backtest, but shadow-mode needs actual order intents with actual order types. If you model market orders in the simulator, you overstate cost on patient strategies; if you model limit fills, you understate cost on urgent ones and overstate fill rate on all of them.

**Specify the order type per signal family and build the fill simulator to model each.** A limit order that doesn't fill in 2 minutes should be canceled and logged as ENTRY_EXPIRED, and the backtest must model that expiration rate.

### 1.3 ATR(5min) at 09:35 is a nonsense estimator

§5.2 uses ATR(5min) to set the target barrier. At 09:35 — five minutes into the session — you have exactly five 1-minute bars. ATR over 5 bars of a just-opened session, on a gapping stock, in the most volatile minutes of the day, is dominated by the opening bar's range. It's not an estimate of anything stable.

Use **prior-session ATR** (e.g., ATR(14) on daily bars, scaled to the intraday horizon by a fixed ratio) or use pre-market range as a volatility proxy. Either is more stable than a 5-bar intraday ATR computed during the opening cross. If you insist on intraday ATR, wait until 09:45 (after the opening range closes) and use the OR range itself as a volatility estimate — at least that's what the ORB strategy already computes.

### 1.4 No data durability plan

The entire project's deliverable is the shadow-mode record: 60+ days of decisions, fills, features, and daily reports. This record lives in a SQLite file on a $5/month VPS with no mentioned backup. If the VPS disk fails, your 14 weeks of accumulated evidence vanish and you restart from nothing.

Add one line to cron: `rclone sync` to any object store (Backblaze B2 is $0.005/GB/month — you'll spend $0.02/month) nightly after `eod_reconcile.py`. Also back up `research.duckdb` and the `models/` directory. This is 10 minutes of work and it protects the most expensive asset in the project (your time, measured in weeks).

### 1.5 The deploy gate's correlation check is self-defeating early on

§6.7 requires the candidate model's predictions to correlate ≥ 0.6 with the incumbent. In the early iterations, the incumbent is a rules baseline that was the best of your first attempt. Requiring high correlation with it means the gate actively prevents the model from learning something the rules missed. This constraint makes sense in steady-state production (where a wildly different model is a bug signal). It makes no sense during the first 6 months, when the whole point is that the model might find structure the rules didn't.

Remove the correlation check until you have a model that has cleared shadow-mode validation. Before that, the other four checks (OOS Sharpe, PSI, trade count, human approval) are sufficient.

### 1.6 No mention of Alpaca paper trading as a fill-model validator

§3.1 mentions Alpaca's free paper-trading API in passing ("useful as a second opinion"). This undersells it badly. You should submit every shadow-mode order intent to *both* your own fill simulator and to Alpaca's paper trader, and record both sets of fills. This gives you:

- An independent fill model you didn't write, operated by a broker with actual order-matching logic
- A divergence metric between your simulator and a real (simulated) execution venue
- Evidence about whether your fill assumptions are optimistic *before* shadow mode ends

This is free. Wire it in during week 7 when you build the session runner. If your simulator says you filled at $4.12 and Alpaca's paper trader says $4.18, you have a fill-model calibration error you can fix immediately instead of discovering it via tracking error in week 19.

---

## 2. What the Document Gets Right That the First Review Likely Missed

### 2.1 Pessimistic intrabar resolution for triple-barrier labeling

§5.2's rule — if a single bar touches both the stop and target barriers, assume the stop hit first — is correct and rarely implemented. Most retail backtest frameworks either pick randomly, pick the closer one, or (worst) assume the target hit. This single rule prevents the most common source of inflated backtest results in barrier-based labeling. It's a detail that shows someone has actually debugged a backtest that looked too good.

### 2.2 Logging rejections, not just fills

§10.1's insistence on logging every REJECTED decision with its full feature vector is genuinely unusual and genuinely valuable. The trades you didn't take are more informative than the ones you did: they tell you the counterfactual P&L of your filter, the frequency of each veto reason, and whether the risk engine is too tight or too loose. Most systems log fills and ignore everything upstream. This design makes the scanner and the risk engine debuggable.

### 2.3 The tracking-error gate as a pipeline validator

§8.1's comparison of shadow P&L to backtest-predicted P&L is subtler than it looks. Most shadow-mode plans ask "is shadow profitable?" — which is a one-sided test on a small sample. This design asks "does shadow agree with backtest?" — which is a calibration test that can detect *both* fill-model errors (shadow worse than backtest) *and* data-snooping (backtest better than shadow, which means the in-sample period was overfit). It's the right question.

### 2.4 The 10:55 hard flatten as a structural risk bound

The document correctly identifies that flat-by-11:00 is not just a risk rule but a *design constraint* that makes offline safe, bounds maximum loss duration, and simplifies the entire reliability model. §13.6's acknowledgment that it will "feel wrong, repeatedly" is honest and correct — and the instruction to never add an exception is the only way it works. This is a case where a rigid constraint buys more than a flexible one.

### 2.5 Stating the dollar economics plainly

§1.3's $4.95/day calculation, and the observation that infrastructure costs consume essentially all profit at $1,000, is the kind of arithmetic that most design documents at this stage omit because it's discouraging. Including it, and then structuring the entire project around "this is R&D, not income," is the correct framing and it prevents the most common rationalization failure in retail system design: treating unrealized backtest returns as income.

---

## 3. Are the Kill Gates Realistic?

### Week 1 gate: realistic and correctly placed

The broker capability audit is binary and fast. Either stops can rest at the broker or they can't. Either unattended operation is possible or it isn't. These are answerable in a day of documentation reading, and they genuinely are project-blockers. This gate works.

### Week 2-3 gate: the most important and the most vulnerable to false positives

This is where the whole project lives or dies, and the gate has a multiple-testing problem it doesn't fully address. You're testing five signal families (A-E), each with multiple parameter configurations. If you test 30 total configurations across five families with purged walk-forward on 250 days, the probability that *at least one* shows positive expectancy "across a majority of folds" by chance is non-trivial. §7.4 mentions deflated Sharpe and experiment counting, but those tools are described for the *later* research phase — they need to be applied *at this gate*, explicitly, or the gate will pass noise.

**Concrete risk:** Gap-and-go (signal A) on a favorable 6-month window with 3 parameter choices will produce a rules baseline that looks like it works. You'll pass the gate, invest 4 more weeks, and discover at the week 5 gate that it collapses under full friction. That's the system working as designed — but it's 4 weeks you could have saved by applying the deflated Sharpe correction at week 3.

Mitigation: pre-register the exact rules (not ranges) for each signal family before running the test. One configuration per family, five tests total. If you must search parameters, apply a Bonferroni-level correction at this gate: require positive expectancy in *all* folds, not a majority, and require p < 0.01 on the daily-clustered t-test. Yes, this will fail more often. That's the point.

### Week 4-5 gate: realistic and correctly designed

The "expectancy collapses moving from notebook to full-friction engine" test is the real filter, and it's correctly placed. The double-spread sensitivity test is an excellent addition. This gate will catch most of the false positives that leak through week 3. It works.

### Week 7 gate: the strongest gate in the sequence

"Kill -9 mid-position and verify recovery to a correct flat state" is binary, testable, and automatable. You can write a pytest that does this. It either passes or it doesn't. This is the model for what a kill gate should look like.

### Shadow-mode exit criteria (§8.1): well-designed but possibly infeasible

The criteria are individually reasonable. The problem is statistical power. At 1-2 trades/day over 60 trading days, you get 60-120 trades. With daily clustering, your effective n ≈ 60. A one-sample t-test on daily P&L with n=60 requires a standardized effect size of ~0.36 for 80% power at α=0.05. If your daily P&L standard deviation is $20 (plausible for a $200-500 position system), you need a true expected daily P&L of ~$7 to reliably detect it.

$7/day is *above* the optimistic scenario in §1.3 ($4.95/day). If the true edge is smaller — 0.08R instead of 0.15R, say — then 60 days of shadow mode literally cannot distinguish it from zero, and the "bootstrap CI excluding zero" criterion will fail even if the strategy is real.

This doesn't mean the criteria are wrong. It means you should compute the minimum detectable edge *before starting shadow mode* and decide whether an edge that small is worth pursuing. If the answer is "I need at least 0.15R to justify the infrastructure cost," and 60 days can detect 0.15R with reasonable power, you may need to extend the window. If the answer is "I'd be happy with 0.08R on $10,000 of capital," then 60 days isn't enough and you should plan for 120.

---

## 4. One Specific Recommendation

**Run a pre-mortem power analysis on the shadow-mode exit criteria before writing a line of strategy code.**

Here's why this is the highest-leverage thing you can do:

The entire project funnels toward a single statistical question: "does the bootstrap CI on daily-clustered shadow expectancy exclude zero?" (§8.1). If your sample size can't answer that question for the edge size you're looking for, then no amount of engineering quality, fill-model realism, or risk-engine correctness will save you — you'll spend 20 weeks building a measurement apparatus that isn't sensitive enough to measure the thing.

Concretely, before week 1:

1. **Estimate your daily P&L standard deviation.** Use your backtest data from the week 2-3 research phase — run the baseline rules, compute daily P&L, measure the standard deviation. Call it σ_daily.

2. **Compute the minimum edge your shadow test can detect.** For a block bootstrap with n=60 days, power=0.80, α=0.05, the minimum detectable mean daily P&L ≈ 0.36 × σ_daily (this is the one-sample t-test approximation; the bootstrap will be similar).

3. **Compare that to your break-even.** If break-even is $2.25/day in friction (§4.3) and minimum detectable daily P&L is $7, then you can only detect strategies that clear break-even by $4.75/day — strategies that are solidly profitable, not marginal ones. That might be fine. Or it might mean you'll reject a real but small edge because your test isn't powerful enough.

4. **If the minimum detectable edge is larger than what you'd realistically expect, you have three options:**
   - Extend shadow mode (120 days cuts minimum detectable effect by ~30%)
   - Increase trade frequency (requires more capital or relaxing one-at-a-time)
   - Accept that you're only testing for large edges and will miss small real ones

This takes an afternoon with a spreadsheet. It doesn't require any code. And it's the one analysis that determines whether the entire 20-week plan produces a usable answer or an inconclusive one. Do it before you commit to the timeline, because changing the shadow-mode duration in week 18 is exactly the kind of post-hoc threshold adjustment that §9.3 correctly warns against.

---

## Summary of Findings

| Category | Items |
|---|---|
| **Surviving flaws** | IEX volume used for volume-dependent signals; entry order type unspecified; ATR(5min) computed on 5 bars at session open; no data backup; deploy gate correlation check blocks early improvement; Alpaca paper trading underused |
| **Correctly done, likely missed by first review** | Pessimistic intrabar barrier resolution; rejection logging; tracking-error as pipeline validator; hard flatten as structural constraint; honest dollar arithmetic |
| **Kill gates** | Week 1 and week 7 gates are strong. Week 2-3 gate has a multiple-testing leak. Shadow-mode criteria are well-designed but may lack statistical power for small edges. |
| **Top recommendation** | Power analysis on shadow-mode exit criteria before committing to the timeline. Determines whether 60 days can detect the edge you're looking for. |
