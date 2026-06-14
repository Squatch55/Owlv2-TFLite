# Research Report: Improving a Prop-Firm-Evaluation Futures Bot — Public Repos, SMC Causal Audit, Evidence-Graded Strategies, and Novel ML/Risk Restructuring

*Framing: research/education for the user's own system-development decisions on a FundedNext 25K evaluation bot (MNQ/MES/ES/NQ), not financial advice. Every proposed change must be validated with leakage-free out-of-sample and forward/paper testing under realistic fills before risking capital.*

## TL;DR
- **The binding constraint is insufficient profit generation in weak regimes (timeout failure), not drawdown** (the trailing floor is never breached; failures cluster from ~Feb 6 2026 onward). The highest-value work is therefore **payoff-asymmetry diversification** (add genuine larger-winner/breakout capture to offset the 0.37 payoff ratio), **execution-realism hardening**, and a **consistency-rule-aware pacing controller** — risk-reduction overlays (downsizing, uncertainty gating) attack a part that is already fine.
- **All four named SMC repos are reference-only, not dependency-grade for live signals.** joshyattridge/smart-money-concepts is the only maintained library, but a direct code audit confirms its core `swing_highs_lows` uses a **centered look-ahead window** (default `swing_length=50` → ±50 bars) and every derived structure (BOS/CHoCH, order blocks, liquidity) **repaints**. The other three are a synthetic-index spec scaffold, a "vibe-coded" Go alert scanner, and an MQL5 forex EA — unusable as Python futures dependencies.
- This repaint is **the most likely cause of the user's "SMC improved in-sample AUC but not out-of-sample pass rate"** — a textbook leakage signature. Treat SMC as causally-timestamped, confirmation-shifted **soft event tokens**, never live vetoes, and re-validate every candidate with purged/combinatorial-purged CV + Deflated Sharpe + Probability of Backtest Overfitting before trusting any improvement.

## Key Findings

### 1. Repository evaluation (Deliverable 1)
The four named repos were inspected directly (code, README, license, activity). Only one is a genuine reusable library.

**Repo-by-repo table:**

| Repo | What it is | Lang / License | Maintenance | Causal-correctness | Borrowable | Verdict |
|---|---|---|---|---|---|---|
| **joshyattridge/smart-money-concepts** | Most popular Python SMC indicator library (FVG, swings, BOS/CHoCH, OB, liquidity, sessions, retracements) | Python / **MIT** | Active-ish; latest release **0.0.26, "03 Mar" 2025** ("Dramatically increase OB speed", PR #74); ~1.1k+ stars, ~620+ forks (the live repo page read ~1.1k/621 in Jun 2026; an earlier cache showed ~1.7k/736 — either way, by far the most-used) | **NOT causal** (see §2; centered swing window, FVG uses next candle, BOS/OB confirmed by future bars) | Concept definitions; offline event-discovery logic; `previous_high_low` and `sessions` are the only causal outputs [GitHub](https://github.com/joshyattridge/smart-money-concepts) | **Vendor-and-audit for OFFLINE labeling only; never raw live features** |
| **manuelinfosec/profittown-sniper-smc** | "Quant Spec Document" scaffold for a Step-Index-100 "sniper" bot, $1k→$3k/day, 10–20% risk/trade [github](https://github.com/manuelinfosec/profittown-sniper-smc) | Python / no clear OSS license | ~37 stars, ~14 forks; single contributor [github](https://github.com/manuelinfosec/profittown-sniper-smc) | N/A (spec, not a tested detector); marketing return claims, reckless sizing | Only the OB-confluence scoring *idea* (displacement + unmitigated + sweep + Fib + clean structure) | **Avoid** |
| **Ju571nK/ChartNagari** | Self-hosted ICT/Wyckoff alert scanner, 30+ rules, multi-timeframe, Telegram/Discord, "vibe-coded" with Claude Code | Go + React / **MIT** | ~5 stars; active commits, releases to Apr 2026 | README states HTF zones use confirmed pivots (deliberate lag) — better than joshyattridge but it's an alerting tool, not a Python feature lib | Rule taxonomy, signal-quality scoring (volume/wick/reversal), HTF-context suppression ideas | **Reference-only** |
| **NadirAliOfficial/STAR-EA-v11.20** | MT5 Expert Advisor, 12-scenario engine (OB/FVG/OTE/CRT/Judas/Silver Bullet/AMD), XAUUSD/forex [github](https://github.com/NadirAliOfficial/STAR-EA-v11.20) | MQL5 / **MIT** | ~4 stars, 2 commits; tiny [github](https://github.com/NadirAliOfficial/STAR-EA-v11.20) | Backtest (PF 2.57, 62.5% WR, $225 net on $1k, 12 months single symbol) is tiny-sample, marketing-grade; wrong platform/asset [github](https://github.com/NadirAliOfficial/STAR-EA-v11.20) | Scenario-classification taxonomy as a conceptual checklist | **Avoid as dependency** |

**Best additional repos / libraries:**

| Tool | Role | Fills realistic? | License/maturity | Grade |
|---|---|---|---|---|
| **hftbacktest** (nkaz001) | Queue-position + latency fill simulation, full L2/L3 tick replay [GitHub](https://github.com/nkaz001/hftbacktest) | **Yes (best-in-class queue model)** | Open, active; crypto examples but generalizable [GitHub](https://github.com/nkaz001/hftbacktest) | Reference→vendor for execution-realism methodology |
| **NautilusTrader** | Event-driven, research-to-live parity; [Python](https://python.financial/) L2/L3 fills against actual book levels; L1 uses `FillModel` (e.g. `prob_slippage` moves fill one tick adverse) [NautilusTrader](https://nautilustrader.io/docs/latest/concepts/backtesting/) | **Yes** | Open, Rust+Python, active | **Dependency-grade** for realistic futures backtest |
| **vectorbt** (OSS) / VectorBT PRO | Fast vectorized parameter sweeps [Om Arora](https://omarora.in/quantitative-finance/resources-backtesting-tutorials/) | **No** | Open (PRO commercial) | Research-only |
| **backtrader** | Mature event-driven, brackets/OCO, IB integration [Om Arora](https://omarora.in/quantitative-finance/resources-backtesting-tutorials/) | Partial | Community-maintained | Dependency-grade for order logic |
| **mlfinlab** (hudson-and-thames) | Triple-barrier, meta-labeling, [GitHub](https://github.com/hudson-and-thames/mlfinlab/blob/master/mlfinlab/labeling/labeling.py) purged + combinatorial-purged CV, fractional differentiation, sample-uniqueness | N/A | Open snapshot exists (quantopian/mlfinlab); newer versions commercial; open alt **mlfinpy** | **Methodology-grade** |
| **Microsoft Qlib** | Research orchestration, rolling retrain, RL execution | N/A | Open, active | Dependency-grade pipeline |
| **FinRL / FinRL-Meta** | RL sandbox | Configurable | Open; documented seed instability/overfitting | **Sandbox only** |
| **aangelopoulos/conformal-prediction** | Conformal method reference notebooks | N/A | Open | Reference |
| **DeepLOB** (zcakhaa) | CNN-LSTM LOB price-move model | N/A | Open | Reference |

For OFI / queue-imbalance / micro-price / VPIN there is **no single canonical maintained library**; these are short, well-specified formulas (Cont–Kukanov–Stoikov; Easley–López de Prado–O'Hara) that should be implemented directly on the user's BBO/L1 data.

### 2. SMC/ICT causal-correctness audit (Deliverable 2) — *the user's central suspicion, confirmed*
A direct line-level code audit of `joshyattridge/smart-money-concepts/smc.py` (master, ~987 lines, v0.0.27) confirms pervasive look-ahead:

- **`swing_highs_lows(swing_length=50)` — centered, non-causal.** Code: `swing_length *= 2` then
 `np.where(ohlc["high"] == ohlc["high"].shift(-(swing_length//2)).rolling(swing_length).max(), 1, ...)`. With the default this is `.shift(-50).rolling(100).max()` — bar *i* is flagged a swing only if its high is the max over **~[i−49 … i+50]**, i.e. **50 bars before AND after**, yet the ±1 label and `Level` are **stamped at bar i's own timestamp**. A pivot at *i* cannot be known until ~50 bars later → it repaints. This is the root: everything below inherits it.
- **`fvg()` — 1-bar look-ahead.** Uses `ohlc["low"].shift(-1)` / `ohlc["high"].shift(-1)` (the *next* candle) in both the condition and the `Top`/`Bottom` levels, stamped at the middle candle. Not knowable until candle *i+1* closes.
- **`bos_choch()` — repaints.** The BOS/CHoCH ±1 and `Level` are written at the **past swing index** `last_positions[-2]`, then retained only if a **future** bar (`BrokenIndex`, found by scanning `[i+2:]`) breaks the level. The economically meaningful "this just happened" timestamp is `BrokenIndex`, not where the signal is stored.
- **`ob()` — repaints.** The order-block box (`OB`, `Top`, `Bottom`) is stamped at an **earlier** candle `obIndex` (default `close_index−1`, or the lowest-low bar in the run-up), but the block is only created once a **later** candle's close displaces past the swing high. [GitHub](https://github.com/tpwilo/smc)
- **`liquidity()` / `retracements()`** inherit the swing look-ahead. **Only `previous_high_low` and `sessions` are fully causal.** The forward-pointing index columns (`MitigatedIndex`, `BrokenIndex`, `End`, `Swept`) are deliberately future indices and are only safe if read at *that* future timestamp.

**Generic failure mode and the fix.** Pivot/fractal detection needs N right-side bars to confirm; a swing detected with right-window K is **knowable only at t+K**. The correct causal protocol:
1. **Offline event discovery/labeling** — centered windows are fine (you are describing history for triple-barrier labels).
2. **Live features** — must use closed-bar, backward-only or **confirmation-shifted** values: a swing event timestamped at t+K; an order block timestamped at the displacement/break bar; an FVG at candle *i+1*'s close.
3. **Leakage tests:** point-in-time replay (recompute features using only data ≤ t); compare live vs historical recomputation to detect repainting; embargo windows around event windows (López de Prado's purged CV + embargo); verify a feature's value at t does not change when future bars arrive.

This directly explains the user's signature: **SMC features lifted in-sample AUC but not out-of-sample pass rate** — the canonical leakage fingerprint. Remediation: demote SMC to causally-timestamped soft tokens and re-measure pass30/pass60.

### 3. Evidence-graded strategies (Deliverable 3)

| Strategy | Mechanism | Evidence grade | Key caveats | Fit to this bot |
|---|---|---|---|---|
| **Opening Range Breakout (Zarattini, Barbon & Aziz, SSRN 4729284, Feb 2024)** | 5-min OR breakout [QuantConnect](https://www.quantconnect.com/research/18444/opening-range-breakout-for-stocks-in-play/) on high-relative-volume "Stocks in Play" | **Practitioner/commercial**, large sample (>7,000 stocks, 2016–2023) | Top-20 SIP portfolio reported **">1,600% total net return," "Sharpe ratio of 2.81," "annualized alpha of 36%"** vs S&P 500's ~198%; [SSRN](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4729284) the **15-min ORB achieved 272% / 1.43 Sharpe** (window-specific). Authors sell day-trading education (Aziz) / run Concretum; heavy leverage (TQQQ 3×), idealized fills in some specs | **High strategic value: a payoff-asymmetry IMPROVER** (large winners, trend capture). Re-test as index-futures version with realistic fills |
| **VWAP / anchored-VWAP trend & bands (Zarattini & Aziz, SSRN 4631351, Nov 2023)** | Long above VWAP, short below; [SSRN](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4631351) band mean-reversion | Practitioner | "$25,000 ... would have grown to $192,656 ... 671% return ... maximum drawdown of just 9.4% and a Sharpe Ratio of 2.1" [SSRN](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4631351) on QQQ 2018–2023; same commercial-author/idealized-fill caveats | Anchored-VWAP from session/prior-day open is a robust **causal reference level** — good feature and stop/target anchor |
| **Market intraday momentum (Baltussen, Da, Lammers & Martens, *JFE* 142(1), Oct 2021, pp. 377–403; SSRN 3760365)** | Last-30-min return positively predicted by rest-of-day return; gamma-hedging mechanism [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0304405X21001598) | **Peer-reviewed, robust** | "Using intraday returns on over 60 futures ... 1974–2020, we find strong market intraday momentum everywhere"; strategy "annualized Sharpe ratios between 0.87 and 1.73 at the asset class level"; stronger under high volatility [Super](https://assets.super.so/e46b77e7-ee08-445e-b43f-4ffd88ae0a0e/files/ee7dac49-530b-4950-b5d0-e0b5eee08f2e.pdf) | **Strong fit**: an end-of-day momentum overlay is evidence-graded and directly helps "generate profit on pace" |
| **Intraday U-shape volatility/volume (Andersen–Bollerslev)** | Higher vol at open/close, midday lull [arxiv](https://arxiv.org/pdf/1707.02419) | **Peer-reviewed** | Seasonality, not standalone alpha | Use for time-of-day conditioning only |
| **Liquidity-sweep-reclaim, prior-day H/L, overnight-range, gap-fill, opening-drive fade** | Various reversal/continuation | Mostly practitioner; partial overlap with documented reversal | Require leakage-free validation | Exploratory soft features |

**REMOVE / de-emphasize:** blanket hour bans; hard MTF/SMC vetoes that starve trade density; high-WR-but-near-zero/negative-expectancy filler subfamilies. The user's "raw unconfirmed stack is negative in 2026" and the `momentum_exhaustion_session_extreme` event (avg-R −0.17, avg-PnL +$1.11 standalone over 189 events) are prime pruning candidates — keep only as soft features, not hard signals.

### 4. Novel ML + risk restructuring (Deliverable 4)

**4a — ML beyond the prior-report consensus (honestly graded):**
- **Conformal prediction / conformalized quantile regression** (Vovk; Angelopoulos & Bates, *A Gentle Introduction*, arXiv 2107.07511) — *well-supported, directly applicable.* Distribution-free, coverage-guaranteed [arXiv](https://arxiv.org/abs/2107.07511) accept/skip with a formal abstention ("learning to reject"). Conformalize the strong **FAST_TARGET (AUC≈0.81)** and **TRANSACTION-COMPLETION (≈0.71)** heads; route the weak **WIN head (≈0.59)** to abstention rather than letting it drive sizing.
- **Cost-sensitive / asymmetric-loss learning** — *well-supported.* Penalize large-loss false positives more than missed wins; attacks the 0.37 payoff ratio at the objective level.
- **Calibration (isotonic/Platt, reliability diagrams, Expected Calibration Error)** — *well-supported, prerequisite* for probabilities to legitimately drive sizing.
- **Proper meta-labeling with purged + combinatorial-purged CV, Deflated Sharpe, PBO** (Bailey & López de Prado 2014, SSRN 2460551; Bailey, Borwein, López de Prado & Zhu 2015, SSRN 2326253) — *well-supported, high priority.* CPCV is shown to yield lower PBO and a more reliable Deflated Sharpe than walk-forward; [ScienceDirect](https://www.sciencedirect.com/science/article/abs/pii/S0950705124011110) this is the direct fix for the user's best-of-39–55-candidates selection bias.
- **Bayesian quickest change detection** (Shiryaev; Page's CUSUM; Moustakides optimality; Tartakovsky–Veeravalli) — *well-supported theory, empirical tuning.* Optimal-stopping detection of regime breaks; directly targets the Feb-2026 cliff.
- **Distributional / CVaR-aware RL** for the sizing/pacing controller — *promising but speculative.* Because the true objective is path-dependent certification (not mean reward), a drawdown/CVaR-aware objective is conceptually right, but RL reproducibility is poor (FinRL documents policy instability, single-seed over-reporting, survivorship/overfitting). [arXiv](https://arxiv.org/html/2504.02281v3) Start with a **contextual bandit → conservative offline RL**, not deep on-policy RL.
- **Multi-objective/constrained optimization framed directly on pass30/pass60 under the consistency rule** — *recommended, most direct.*

**4b — Why a high-WR/low-payoff system is fragile.** With avg loss ≈ 2.7× avg win (payoff 0.37), breakeven WR ≈ 1/(1+0.37) ≈ **73%**; the 81.8% WR leaves only **~9 points of cushion**. Negatively-skewed PnL means risk-of-ruin and the drawdown distribution are dominated by rare large losses, so a small WR drop (regime change) or small payoff erosion (cost/fill drift) flips expectancy negative. This is exactly the observed execution-realism gap: harsher NinjaTrader-style fills collapse pass30/pass60 from **30.8%/56.4% → ~10.3%/12.8%**. The consistency-rule trap is a constrained control problem: **too slow → timeout before the target; too fast → one big day inflates the effective target (best_day/0.40) or risks the trailing floor.**

**4c — Risk restructuring (with caveats).**
- **Sizing:** Full Kelly with a 0.37 payoff implies absurd fractions and catastrophic drawdowns under estimation error (Browne–Whitt's "catastrophic outcomes"; Ziemba advocates fractional Kelly with dynamic adjustment; [Medium](https://medium.com/@tmapendembe_28659/kelly-criterion-vs-fixed-fractional-which-risk-model-maximizes-long-term-growth-972ecb606e6c) half-Kelly retains ~75% of growth [JournalPlus](https://journalplus.co/learn/guides/kelly-criterion-guide/) at far lower variance). Use **heavy fractional Kelly (¼ or less) or fixed-fractional, hard-capped** — and note account-size constraints usually bind first. But sizing is **not the main lever** here (the drawdown floor is never breached); profit generation is.
- **Exits:** There is real evidence that tightening stops / adding time-stops can make a high-WR system "**prettier but worse**" by cutting the rare large winners it depends on. Evaluate any exit change by its effect on the **payoff ratio**, not just WR.
- **Consistency-rule-aware pacing:** treat daily profit as an **optimal-stopping / daily-budget** problem — ease off once near the ≤40%-best-day cap; press (within risk limits) in weak regimes to avoid timeout.
- **Validation:** replace day-shuffle Monte Carlo — which is **over-optimistic** (control mc_pass30 0.66 vs audited 0.308) because shuffling day order destroys the regime clustering that is the real killer — with **regime-aware block bootstrap** and **paired / Deflated significance testing**. The "best" hybrid's 5-point pass30 gain (35.9% vs 30.8%) on a tiny selected sample is plausibly multiple-testing noise until it survives deflation and split-half stability.

## Details — Prioritized roadmap (add / modify / remove)

**Tier 1 — highest leverage, attacks the binding constraint (timeout in weak regimes):**
1. **Add a payoff-asymmetry diversifier**: an evidence-graded breakout/trend module — ORB-style entries and/or an end-of-day intraday-momentum overlay (Baltussen et al. 2021) — to inject larger winners that raise the 0.37 payoff ratio and generate profit on pace. Validate on MNQ/MES under harsh fills. **Promote only if** it lifts pass60 in the Feb-2026+ timeout cluster while keeping max DD ≤ ~$732 and pushing payoff ratio above ~0.5.
2. **Make harsh fills the PRIMARY backtest** (stop-market 2-tick adverse, limit targets requiring trade-through, adverse market/EOD exits). Any candidate must survive this, not the optimistic model. **Buy MBP-10/MBO depth + trade prints + paper/live fill telemetry, plus 4–6 years of history, before buying any generic indicator library.**
3. **Consistency-rule-aware pacing controller** (contextual bandit → conservative offline RL) optimizing P(certification), with the ≤40%-best-day cap and trailing floor as explicit constraints.

**Tier 2:**
4. **Fix validation infrastructure**: purged + combinatorial-purged CV, Deflated Sharpe, PBO; regime-aware block bootstrap replacing day-shuffle MC. Gate ALL candidate selection on these.
5. **Conformal accept/abstain gate** driven by the FAST_TARGET / TRANSACTION-COMPLETION heads; abstain on weak-WIN-head cases.
6. **Regime gate / quickest-change detection** for the Feb-2026-type cliff.

**Tier 3:**
7. Convert SMC/ICT to **causally-timestamped, confirmation-shifted soft event tokens** (offline-discovered), never live vetoes.
8. **Cost-sensitive / asymmetric loss** targeting large-loss false positives.

**Remove / de-emphasize:** blanket hour bans; hard MTF/SMC vetoes that starve density; near-zero-expectancy filler subfamilies; the standalone `momentum_exhaustion_session_extreme` event as a hard signal.

## Recommendations (staged, with thresholds)
1. **First — leakage audit.** Run existing SMC features through point-in-time replay. **If** removing live SMC features does not hurt out-of-sample pass rate, they were leaking (very likely given the centered-window code) → demote to offline tokens and re-measure pass30/pass60.
2. **Second — payoff diversifier under harsh fills.** Build the breakout/momentum module; promote only if it improves timeout-cluster pass60 at max DD ≤ ~$732 and raises the payoff ratio.
3. **Third — selection-bias-robust validation.** Require any candidate's pass30 improvement to survive Deflated-Sharpe correction for the ~39–55 trials and be stable across split-halves and a regime-aware block bootstrap.
4. **Fourth — conformal gate + pacing controller.** Benchmark: higher pass30/pass60 at equal-or-lower trade density and drawdown.
5. Reserve diffusion / autoregressive / Neural-SDE / world-model approaches for **scenario generation, stress-testing, and simulator/RL support**, not first-line live signals.
6. Validate every change with leakage-free OOS + forward/paper testing under realistic fills before risking capital.

## Caveats
- **Intraday signal-to-noise is very low**; most online "edges" are over-fit or marketing. The ORB/VWAP headline numbers (1,600%/2.81 Sharpe; 671%/2.1 Sharpe) come from **practitioner/commercial authors using leverage and idealized fills** — do not treat them as established facts; the 5-min-vs-15-min ORB gap (1,600%+ vs 272%) shows window-sensitivity consistent with over-fitting.
- The user's "best" candidate's 5-point pass30 gain is plausibly multiple-testing noise on a small, hard-regime-split, selected sample.
- **RL-for-trading is reproducibility-fragile** (FinRL seed instability, information-leakage risk); use as sandbox/controller research only.
- **Tree ensembles (XGBoost/LightGBM/CatBoost) still beat deep nets on tabular financial features**, and the DLinear result ("Are Transformers Effective for Time Series Forecasting?", Zeng et al., AAAI 2023) shows simple linear baselines often beat Transformers on time series [AAAI](https://ojs.aaai.org/index.php/AAAI/article/view/26317/26089) — do not over-engineer architecture; the tractable signal here is in path-speed and transaction-completion, not win/lose.
- Where this report cites star/fork counts and dates, GitHub readings differed slightly between fetches (e.g., ~1.1k vs ~1.7k stars for joshyattridge); the qualitative conclusion — it is the dominant Python SMC library and it repaints — is unaffected.
- This is research/education for the user's own informed decisions, not financial advice.