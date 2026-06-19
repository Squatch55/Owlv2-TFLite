# FundedNext 25K Futures Bot: Root Causes and a Concrete Improvement Plan

## The Core Problem Is Not Signal Quality — It's Three Compounding Constraints

The control baseline (pass10 55.77%, pass20 98.08%, R:R 1.41) proves the underlying strategy logic works. Applying the hard $50 risk cap collapses pass10 to 0% and pass20 to 5.77%, but R:R *improves* to 2.51 — the constraint is already acting as a quality filter, selecting only high-conviction setups. The problem is not trade quality at $50; it is that the bot generates only 1.1–1.7 qualifying trades per day at $50 risk, and the challenge requires roughly 2.5R/day to pass in 10 days. That throughput gap is the root cause, and it cannot be closed by feature engineering alone — 38 consecutive causal ML expansions all failed to improve pass10/pass20, which is strong evidence the bottleneck is structural (stop geometry + trade frequency), not predictive.

Three separable sub-problems must all be addressed:

- **(A) Stop geometry**: can structurally tighter stops be found without sacrificing win rate?
- **(B) Trade frequency**: can more qualifying setups be generated per day at $50 risk?
- **(C) Infrastructure fidelity**: are current test results measuring the right thing?

Solving only A or only B is insufficient. The math is unforgiving: at 1.85 trades/day and 52–60% win rate, the gross reward/risk needed before costs is 2.92–4.93 — a range that demands both tighter stops (so more trades qualify) and more setups per day (so the compounding can happen).

| Metric | Control | Hard $50 | Gap |
|---|---|---|---|
| Pass10 | 55.77% | 0% | Critical |
| Pass20 | 98.08% | 5.77% | Critical |
| Max DD | \$225 | \$121 | Better |
| R:R | 1.41 | 2.51 | Better |
| R/day needed | — | ~2.5R | Unmet |

There is also a second, underappreciated gap: the stacked-backtest control (55.77% pass10) is inflated by same-timestamp position stacking that affects 50 of 52 starts and cannot be reproduced by the live single-position engine. The honest proxy-free baseline is 7.69% pass10. This means the real live-trading starting point is much lower than the headline control number suggests.

---

## Section 1 — Infrastructure Fixes: Do These Before Any ML Work

Every simulation result produced under the current configuration is partially invalidated. Fix these first; they are cheap and they unblock everything else.

**Live-risk parameter mismatch.** `strict_under_month` uses \$150 MNQ / \$75 MES instead of the intended \$40–\$50 target. In NinjaScript, locate the `MaxLossPerTrade` or equivalent parameter block in the `strict_under_month` branch and set it to `50` for MNQ and `25` for MES (half-sized). Until this is corrected, every live-mode shadow run is testing a bot with 3× the intended risk — results from those runs cannot be compared to $50-risk backtests.

**Same-timestamp coalescing.** NinjaTrader's backtest fill model processes only at bar close and allows multiple positions at the same timestamp [1]; the live engine cannot reproduce this. The fix is to add deterministic signal coalescing: when multiple signals fire at the same bar, select exactly one using a priority rule (e.g., highest predicted confidence, or earliest signal in the queue). Set `EntriesPerDirection = 1` and `EntryHandling = UniqueEntries` in the strategy properties. This will lower the stacked-backtest pass10 from 55.77% toward the honest proxy-free baseline, but it will also make the number trustworthy.

**Session cutoff correction.** The legacy backtest used 15:45/15:55 ET force-flat; production uses 16:50. The gap is roughly 55–65 minutes of live RTH trading per day. At 1.85 trades/day across a ~6.5-hour RTH session, the lost window represents approximately 15–20% of daily setup opportunities. Confirm the DST-aware 16:50 cutoff is compiled into the current NinjaScript and that the backtest `SessionTemplate` matches production exactly.

**Fill model: MIT vs. stop-limit for tight stops.** At \$50 risk on MNQ (≈4 ticks at \$0.50/tick), a stop-limit order triggers a limit order when the stop price is touched — if the market gaps through the limit, the order may not fill at all [from NinjaTrader order type documentation]. A Market-If-Touched (MIT) order triggers a market order on touch, guaranteeing a fill at the cost of one additional tick of slippage in fast markets. For a 4-tick stop budget, the difference between "fill guaranteed at market" and "limit may not fill" is material. **Recommendation: use MIT for stop exits in the $50-risk regime.** The fill guarantee is worth the marginal slippage compared to the risk of a stop-limit that misses entirely and lets a loss run. NinjaTrader's `OnExecutionUpdate` is called after `OnOrderUpdate` and handles partial fills [2][3] — the existing bid/ask capture infrastructure should log realized slippage per exit so you can measure the actual MIT vs. limit-fill cost difference over 30+ live observations.

**OOF minimum sample gate.** The premarket candidate was rejected at 9 OOF events. That rejection was correct — 9 binary outcomes cannot distinguish a 10% improvement from noise. Establish a hard rule: no strategy variant is accepted or rejected until it has at least 30 OOF events (50 preferred). With 52 total starts, this means a new variant needs to be tested across at least 30 non-overlapping challenge windows before any promotion decision. This gate prevents both false positives (accepting noise as signal) and false negatives (rejecting the premarket direction prematurely).

---

## Section 2 — Stop Geometry: The Highest-Leverage Engineering Problem

The $50 cap at 4 ticks on MNQ leaves almost no room for ATR-based or fixed-offset stops. Every tick of stop budget wasted on placement imprecision is a trade that gets stopped out on a valid setup. The goal is to place stops at *structural invalidation points* rather than fixed distances, so the 4-tick budget buys more protection per dollar of risk.

**Order flow imbalance for stop anchoring.** Order book imbalance (OBI) — the ratio of buy-side to sell-side resting volume at the top N levels — predicts short-horizon price direction and identifies absorption zones where large resting orders defend a price. NinjaTrader's `OnMarketDepth` event [4] provides the Level II data stream needed to compute OBI in real time. A stop placed just below a large bid absorption cluster (where buy-side resting volume is 3–5× sell-side) is structurally tighter than one placed at ATR × 0.5, because the cluster itself acts as the invalidation signal: if price breaks through it, the setup thesis is genuinely wrong. This can recover 1–2 ticks of stop budget compared to a fixed offset. Caveat: `OnMarketDepth` is CPU-intensive as a real-time stream and cannot be backtested on historical bar data — it requires shadow-mode live validation before production use.

**VPOC-anchored stops.** Volume Profile Point of Control (VPOC) and Value Area Low/High (VAL/VAH) are price levels where the highest volume traded. Price tends to rotate around VPOC and rarely re-enters high-volume nodes without a trend reversal — which makes them natural stop-anchor points. A stop placed 1 tick beyond VAL on a long entry is structurally valid: if VAL breaks, the value area has shifted and the trade thesis is wrong. On MNQ 5-minute data, VAL/VAH can be computed from the prior session's volume distribution and updated intraday. This approach is compatible with NinjaTrader's existing bar data and does not require Level II streaming.

**Bid/ask-side entry precision.** The existing bid/ask capture infrastructure can be used to place entries at the ask side of a support cluster rather than the mid or a fixed offset. On a long entry, entering at ask (rather than ask + 1 tick) recovers half a tick of stop budget. At 4-tick total budget, half a tick is 12.5% of the risk envelope.

**Asymmetric architecture test.** The current R:R at hard $50 is 2.51, implying the bot is already finding high-R setups. The question is whether a tighter stop (e.g., 3 ticks / \$37.50 on MNQ) paired with a proportionally larger target (e.g., 7.5 ticks / \$93.75, maintaining ~2.5R) would pass the challenge faster by reducing stop-out frequency on valid setups. This is worth testing as a discrete experiment: run the same signal set with a 3-tick stop / 7.5-tick target and compare pass10/pass20 against the 4-tick baseline. The hypothesis is that the tighter stop filters out more marginal entries, increasing win rate enough to offset the reduced number of qualifying trades.

**Volatility-conditioned stop sizing.** Rather than a fixed \$50 hard cap in all conditions, implement a regime-gated stop: use \$40–\$45 in low-volatility regimes (where the 4-tick budget is structurally adequate) and hold at \$50 in normal regimes. Do not expand stops in high-volatility regimes — the challenge's max drawdown constraint makes that dangerous. The VVG classifier validated on 947 days of MNQ 5-minute data (2021–2025) [5] provides a ready-made regime signal for this gating logic.

---

## Section 3 — Regime Detection: The Architectural Layer the 38 Expansions Were Missing

The most likely explanation for 38 consecutive ML expansion failures is regime mixing: a feature that predicts direction in trending markets anti-predicts in ranging markets. When both regimes are in the training set without conditioning, the net signal averages toward zero. This is the stationarity problem in financial ML — the joint distribution of features and outcomes shifts across regimes, so a model trained on the pooled data learns the average of two incompatible functions.

**HMM as the primary regime filter.** A 2–3 state Hidden Markov Model trained on MNQ-specific features (realized volatility, volume, price momentum, bid/ask spread) is the correct first implementation. HMM outperforms GMM and agglomerative clustering for S&P 500 futures regime detection with steadier state continuity [6]. The VVG classifier [5] — validated specifically on MNQ 5-minute data — provides an instrument-specific alternative or cross-validation reference. Implement as follows:

- **States**: 2 states minimum (trending / ranging); 3 states (trending / ranging / high-volatility) if data supports it. Do not use more than 3 — with 52 challenge runs, a 4+ state HMM will overfit the transition matrix.
- **Features**: rolling 20-bar realized volatility, 20-bar volume z-score, 10-bar price momentum, bid/ask spread (from existing capture). Keep the feature count to 4–5.
- **Training data**: use the full MNQ historical bar data available (not just the 52 challenge runs). The HMM learns market structure, not challenge-specific behavior, so more bar history is better.
- **Integration**: the HMM regime label (0 = trending, 1 = ranging, 2 = high-vol) becomes a categorical gate in the existing signal pipeline. Momentum/breakout entries are enabled in state 0 only; mean-reversion entries (if added) in state 1 only; both disabled or position-halved in state 2.
- **Validation**: hold out 2024–2025 data for out-of-sample regime label evaluation. Use Bayesian HMM (with Dirichlet priors on transition probabilities) to regularize against sparse state transitions in the training set — Bayesian methods naturally incorporate regularization through priors, reducing overfitting risk on small datasets.

**Why this addresses the 38-expansion failure.** The prior ML expansions added features to a model that was pooling across regimes. Adding a regime gate does not require retraining the existing signal model from scratch — it adds a conditional layer on top. The existing sequence model (AUC 0.75 for pass10, 0.87 for pass20) likely has latent regime sensitivity that is being diluted. Re-running feature importance analysis separately within each HMM state will show which features are predictive in which regime — this is the regime-conditional feature importance analysis that was missing from all 38 prior expansions.

**February underperformance as a regime signal.** Pass10 drops to 30–40% in February vs. 50–71% in other months. February is historically a period of lower directional follow-through in equity index futures. This is consistent with the bot's momentum-oriented strategy underperforming in a ranging/choppy regime. If the HMM correctly identifies February's dominant regime as ranging, and the bot reduces position size or disables momentum entries in that state, February pass10 should improve toward the overall average.

---

## Section 4 — ML Architecture: What to Build, What to Skip

### Tier 1 — Build Now (Low Complexity, High Payoff)

**GBT with walk-forward CV and SHAP.** XGBoost or LightGBM on the existing tabular feature set is more robust than deep learning for this problem: ~52 challenge starts × ~5–15 features is a tabular small-data problem where tree ensembles consistently outperform neural networks [from SVR/XGBoost small-dataset validation evidence]. The critical implementation detail is time-series cross-validation: use anchored walk-forward splits (fix training start at the earliest available date, expand the window forward one month at a time, test on the next month). Do not use random train/test splits — that introduces look-ahead bias on a time-ordered dataset. SHAP values on the trained GBT will show which features drive pass/fail predictions within each regime state, directly informing which signals to add or remove.

**Calibrated probability thresholding.** Instead of binary trade/no-trade signals, train the GBT to output a calibrated pass-probability score (use Platt scaling or isotonic regression on the raw GBT output). Then execute only trades in the top 30% by predicted pass-probability. This directly addresses the "fewer but better trades" dynamic at $50 risk: the bot already selects higher-R trades under the $50 constraint (R:R 2.51 vs. 1.41 at control), so a probability threshold will amplify this quality filter. The tradeoff is further reduction in trade frequency — which is why this tier-1 item must be paired with the frequency expansion work in Section 5.

**Bayesian hyperparameter optimization.** Replace any grid or random search with Optuna (or equivalent) for all future parameter tuning. Bayesian optimization with priors naturally regularizes against overfitting on small datasets by concentrating search effort in regions with evidence of improvement rather than sampling uniformly. The 0/158 parameter sweep result (no hits on the R:R stabilization target) is partly a consequence of grid search inefficiency on a sparse reward surface.

### Tier 2 — Build in Month 2–3 (Medium Complexity)

**Path signatures as feature extractors.** Path signature features — specifically, the iterated integrals of a price/volume path up to truncation order 4 — provide a mathematically grounded, universal feature extraction method for sequential financial data [7][8]. At order 4 truncation, a 341-dimensional signature vector captures the full statistical structure of a price path including curvature, cross-correlations between dimensions, and direction-of-travel information that scalar indicators miss. For the challenge context, compute the signature of the last N bars' (price, volume, bid/ask imbalance) path as input features to the GBT. This is architecturally distinct from all 38 prior causal ML expansions and addresses the path-dependent nature of the challenge (pass10 requires consistent early-day performance, not just good individual trades).

**Decision Transformer — 6-month roadmap, not now.** The Decision Transformer models trajectories autoregressively and can stitch together winning subsequences from different training paths, conditioned on a target return. For the challenge, the input would be (last N bars of features + current challenge P&L state + days remaining) and the output would be trade/no-trade + sizing. This is architecturally well-suited to the challenge's path-dependent pass/fail structure. However, it requires a minimum of several hundred challenge simulation paths for meaningful training — with 52 current runs, training a Decision Transformer would overfit severely. Flag for implementation when the simulation path count reaches 300–500.

### Tier 3 — Not Recommended

**Graph Neural Networks.** GNNs are useful for cross-asset correlation modeling but MNQ/MES is a one- or two-asset problem. The added complexity is not justified unless a specific cross-asset signal (e.g., NQ/ES spread divergence as a directional predictor) is identified that a simpler correlation feature cannot capture.

**Deep RL (PPO/SAC) for sizing.** The challenge reward is sparse (pass/fail at day 10/20/30) and the action space is continuous. Reward shaping for sparse delayed outcomes is an open research problem. Without a high-fidelity simulation environment that accurately models fill latency, slippage, and the challenge's trailing drawdown rule, a RL agent will overfit to the simulator's artifacts. Not recommended until the infrastructure fidelity issues in Section 1 are fully resolved.

### Overfitting Guardrails (Mandatory for All Tiers)

- **Walk-forward only**: anchored expanding window, never random split or sliding window that discards early data.
- **30-event minimum OOF gate**: no variant accepted or rejected below this threshold.
- **Feature count discipline**: no more than 10–15 features per model given current data volume. Each additional feature beyond 10 requires a documented causal hypothesis (not just correlation).
- **Regularization by model type**: L1/L2 for linear models; `max_depth ≤ 4`, `min_child_weight ≥ 5` for GBT; dropout + early stopping for any neural component.
- **No-harm gate**: any new variant must pass pass20 ≥ 98.08%, pass30 = 100%, max DD ≤ \$225, bust = 0%, and 5th-percentile bootstrapped pass10 not below the control baseline before promotion.

---

## Section 5 — Trade Frequency: The Throughput Problem

Even with perfect stop geometry and a regime gate, the bot needs more qualifying setups per day to hit 2.5R/day. At 1.85 trades/day, even a 60% win rate and 2.5R target produces only ~1.1R/day net — below the challenge requirement. The frequency gap is approximately 1.2 additional qualifying trades/day needed.

**ETH session activation (pre-market hours).** The shadow monitoring infrastructure for 18:00–09:30 ET is already built and orders are disabled pending validation. The premarket candidate improved pass10 to 9.62% — the rejection at 9 OOF events was correct, but the direction is right. To activate safely:

1. Collect 30+ broker-observed ETH signals with full fill telemetry (bid/ask at entry, slippage, partial fills) before enabling live orders.
2. Apply reduced position sizing during ETH: MNQ liquidity is materially lower during 18:00–20:00 and 02:00–07:00 ET. The 07:00–09:30 pre-RTH window has higher liquidity and is the priority target.
3. Add a spread filter: if the bid/ask spread exceeds 2 ticks, skip the setup. At \$50 risk, a 2-tick spread consumes 50% of the stop budget before the trade even opens.
4. Apply the same HMM regime gate from Section 3 — pre-market sessions have their own regime characteristics and should not inherit RTH regime labels directly.

**Multi-timeframe trend filter.** Adding a 15-minute or 30-minute trend bias as a gating condition does not reduce setup frequency — it filters *bad* setups (counter-trend entries in trending regimes) while leaving good ones intact. A simple 15-min EMA direction (price above/below 15-min EMA(20)) as an entry filter is testable with existing bar data and has minimal overfitting risk given its simplicity.

**Mean-reversion complement.** If the current strategy is primarily momentum/breakout, a mean-reversion strategy for ranging regimes (gated by HMM state 1) can double effective setup frequency without increasing per-trade risk. The key constraint: the mean-reversion and momentum strategies must be structurally uncorrelated — they should not both be long or short at the same time. Use the HMM regime state as a hard gate (not a soft blend) to enforce this.

**FundedNext consistency rule calibration.** The Legacy Challenge enforces a 40% consistency rule: daily profit cannot exceed 40% of the total profit target. For the \$25K Legacy Challenge with a \$1,250 profit target, this caps daily profit at \$500 [from FundedNext documentation]. Verify the bot has a `DailyProfitCap` parameter set to \$500 and that it halts new entries (not just position sizing) when the cap is reached intraday. Overshooting this rule disqualifies the attempt regardless of total P&L. The no-daily-loss-limit structure (confirmed for Legacy/Rapid challenge types) is a genuine structural advantage: the bot can accept larger intraday drawdowns as long as it recovers, which is a different risk profile than most prop firm challenge bots that must also defend a daily loss floor.

---

## Section 6 — Prioritized Implementation Roadmap

| Phase | Action | Why This Order | Overfitting Risk | Effort |
|---|---|---|---|---|
| 1 (Week 1–2) | Fix `strict_under_month` (\$150→\$50 MNQ), fix session cutoff to 16:50 | All current live results are invalid without this | None | Low |
| 1 (Week 1–2) | Add deterministic signal coalescing (`EntriesPerDirection=1`) | Eliminates the 50/52-start stacking artifact | None | Low |
| 1 (Week 1–2) | Establish 30-event OOF minimum gate | Prevents false accept/reject of all future variants | None | Low |
| 2 (Week 2–4) | Implement 2–3 state HMM regime detector on MNQ bar data | Prerequisite for regime-conditional feature importance | Medium | Medium |
| 2 (Week 2–4) | VPOC-anchored stop placement (bar data, no Level II needed) | Tightest stop geometry achievable without streaming infra | Low | Medium |
| 3 (Month 2) | GBT with anchored walk-forward CV + SHAP, regime-conditional | Signal quality improvement with interpretability | Medium | Medium |
| 3 (Month 2) | ETH pre-market activation after 30+ broker-observed signals | Frequency +20–40% if pre-market edge holds | Low | Medium |
| 3 (Month 2) | Daily profit cap at \$500, verify consistency rule compliance | Prevents disqualification on strong days | None | Low |
| 4 (Month 3) | Path signature features (order 4) as GBT input | Untested direction, architecturally distinct from 38 failures | Medium | Medium |
| 4 (Month 3) | Calibrated probability thresholding (top 30% setups) | Quality filter on top of regime gate | Low | Low |
| 4 (Month 3) | OBI via `OnMarketDepth` in shadow mode | Stop placement improvement, requires live validation | Low | High |
| 5 (6mo+) | Decision Transformer (requires 300–500 simulation paths) | Path-dependent optimization, premature now | High | High |

**Sequencing rationale**: Infrastructure fixes come first because all current results are partially invalidated. The HMM regime gate comes before new ML models because it determines which features are relevant in which context — training a GBT on regime-mixed data repeats the failure mode of the 38 prior expansions. New signal architectures (path signatures, OBI) come after the regime gate is validated, so their features are evaluated in the correct conditional context. The Decision Transformer is last because it requires a large simulation corpus that does not yet exist.

---

## Section 7 — Three Non-Obvious Findings

**The R:R improvement at \$50 is a selection effect, not a failure.** The jump from R:R 1.41 to 2.51 when the $50 cap is applied means the constraint is already filtering out low-conviction setups. The bot is not taking bad trades at $50 — it is taking too few trades. This reframes the problem: the goal is not to improve the quality of the $50 trades (they are already better than the control trades), but to find more setups that structurally qualify for a 4-tick stop. Every improvement in stop geometry directly expands the set of qualifying setups, which is the primary frequency lever.

**The 38 ML expansion failures are diagnostic, not just negative.** 38 consecutive failures to move pass10/pass20 with causal ML features is strong evidence that the problem is not in the feature space — it is in the model architecture. Specifically: applying features without regime conditioning means the model is averaging across incompatible market states. A feature that predicts direction in trending regimes may anti-predict in ranging regimes; the net effect across both is near zero, which is exactly what was observed. The fix is not more features — it is regime-conditional feature importance, which none of the 38 expansions implemented.

**The no-daily-loss-limit rule is an underexploited structural advantage.** Most prop firm challenge bots are constrained by both a daily loss limit and a max drawdown limit, which forces them to run a defensive intraday profile even when they are ahead. The FundedNext Legacy Challenge has no daily loss limit — only a trailing maximum drawdown [from FundedNext documentation]. This means the bot can accept a larger intraday drawdown swing as long as it recovers within the session, which is a materially different risk profile. A strategy that takes 2–3 aggressive entries early in a session (accepting a possible \$150–\$200 intraday drawdown) and recovers to \$100+ net by close is legal under Legacy rules but would be disqualified under a standard 2% daily loss limit. This structural freedom should be explicitly modeled in the challenge path simulator.

## References

[1] Discrepancies: Real-Time vs Backtest. https://ninjatrader.com/support/helpGuides/nt8/discrepancies_real-time_vs_bac.htm
[2] OnExecutionUpdate() - Strategy - NinjaTrader 8. https://ninjatrader.com/support/helpGuides/nt8/onexecutionupdate.htm
[3] OnOrderUpdate() - Strategy - NinjaTrader 8. https://ninjatrader.com/support/helpGuides/nt8/onorderupdate.htm
[4] Order Flow Market Depth Map - NinjaTrader 8. https://ninjatrader.com/support/helpGuides/nt8/order_flow_market_depth_map.htm
[5] A Validated Volatility-Volume-Gap Classifier for Regime ... - arXiv. https://arxiv.org/pdf/2605.11423
[6] Market regime detection using Statistical and ML based approaches. https://developers.lseg.com/en/article-catalog/article/market-regime-detection
[7] Path Signatures as Universal Feature Extractors for Limit Order Book. https://papers.ssrn.com/sol3/Delivery.cfm/6635378.pdf?abstractid=6635378&mirid=1
[8] Double-Execution Strategies using Path Signatures. https://ora.ox.ac.uk/objects/uuid:0a493abb-ab8c-4311-bf62-63d71cf487f9/files/rs4655h91w