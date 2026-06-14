# Executive Summary

The current *FundedNext* 25K futures bot has a strong win-rate (~60%) and profit factor, but large occasional losses cause it to miss certification.  Our review finds that *risk management leaks* (oversized losses, static stops, fixed sizing) and limited market awareness (no macro or orderflow features) are key failure points.  We propose a combined approach of **advanced ML signals** and **robust risk controls**.  On the ML side, state-of-the-art techniques (e.g. Transformers, RL with risk constraints, diffusion-based denoising) can extract subtle patterns and incorporate long-range dependencies.  For risk, methods like distributional RL (optimizing CVaR) and explicit drawdown limits can enforce safety.  We also advocate integrating industry “Smart Money Concept” feature libraries (order-blocks, FVG, liquidity sweeps) from open-source repos to enrich inputs.  A detailed roadmap (tables below) prioritizes: acquiring finer-grained data (1-sec bars, Top-of-Book quotes), engineering microstructure features, training/validating ML models and RL agents under realistic fills, and gradually upgrading the backtest risk module.  Each proposal includes required data, expected benefits, evaluation metrics (pass30/pass60, drawdown, Sharpe, drawdown-based risk metrics) and compute estimates. 

Empirical and literature evidence guide our plan.  For example, diffusion models have been shown to **denoise** noisy price series, leading to significantly more accurate return predictions and profitable signals.  Similarly, risk-aware RL frameworks like DeepScalper employ multi-timescale embeddings and a “risk auxiliary task” to balance profit vs loss.  Our analysis also draws on a recent RL trading study: “more historical data worsened performance” due to overfitting, underscoring the need for careful feature selection.  In summary, we recommend a hybrid approach: enhance features (market microstructure + SMC indicators), leverage powerful ML architectures (with cautious evaluation), and tighten risk (dynamic stops, CVaR/RL safeguards).  The subsequent sections detail the diagnostic analysis, literature survey, repository review, data/instrument plans, and a concrete implementation roadmap with ablation experiments.

## Current Strategy Analysis

- **Performance Summary:** The existing control strategy (C_CEIL) yields a ~30.8% chance to hit $1,250 within 30 days and ~56.4% within 60 days (no “bust” events).  Win-rate is ~60%, profit factor ~1.66, but most profits come from small wins.  Critical drawdowns occur from a few large losses (e.g. –$300+, >4× average loss).  These few outlier trades dominate maxDD and kill the funded run. 

- **Loss Drivers:**  Trade-level review shows losses spike when volatile swings trigger stops.  For example, fading a strong intraday move has sometimes backfired (stop-out beyond the $200-$250 daily kill).  Static stops (fixed tick count) and fixed sizing (same contracts) mean risk per trade isn’t adapting to volatility or account state.  There is no mechanism to *scale in/out* or adjust trade size after runups or drawdowns.  Moreover, no regime filter is applied (e.g. economic news events can drive large moves that our bot currently doesn’t avoid). 

- **Regime Failures:**  Certain days (e.g. high-impact news or overnight gaps) repeatedly overwhelm the strategy.  Also, the flat-by-16:10 ET rule and data ending at 16:00 may leave open positions un-closed into the gap period.  In practice the bot was forced flat early, which mismatches the FundedNext requirement.  This “session mismatch” increases effective volatility on open.  Empirically, attempts to extend sessions (16:30/16:50 ET) failed (due to data limits), so alternate risk measures are needed.  

- **Risk Management Gaps:**  Although a daily kill-switch is in place (–$200), the current sizing and stop logic still allow occasional larger losses (up to ~$732 maxDD in tests).  In tests, adding a tighter kill-switch reduced drawdowns by ~50% at the cost of slower target reach (pass30 fell from 30.8%→17.9%).  This tradeoff suggests we need smarter, not just stricter, risk rules.  For instance, volatility-based stops or partial exits could prevent one large loss without killing the run too early.  Studies show disciplined stop-loss usage can dramatically reduce losses (e.g. survival-analysis of traders found stop-loss rules reversed reluctance to realize losses). 

In sum, the strategy’s *signal generation* is decent, but *execution risk* is uncontrolled.  The high win-rate but low R:R per trade implies any improvement in cutting big losses will greatly boost net expectancy.  We will therefore focus on adding: (a) better signal features (via ML + SMC indicators), and (b) adaptive risk controls (dynamic stops, portfolio-level constraints).

## Academic/Industry Survey: ML & Risk in Trading

### Risk-Aware Reinforcement Learning

- **Deep RL Adaptability:**  DRL can learn dynamic strategies in complex markets.  Liu & Tian (2024) review DRL in HFT and note its *adaptive decision-making* strengths but warn of “insufficient real-time performance, data sparsity, model overfitting, and risk management complexity” in practice.  They suggest enhancing algorithmic structures and risk modules to improve HFT outcomes.  

- **Risk Constraints & CVaR:**  Standard RL optimizes expected reward, which ignores tail risk.  Distributional RL can incorporate risk measures: for example, a recent NeurIPS paper extends the distributional Bellman operator to **learn CVaR-optimized policies**.  In practice, this means an agent can be tuned to maximize worst-case returns (e.g. 5% CVaR) rather than mean profit.  Such risk-sensitive RL (and *risk-distortion layers*) are promising for trading, where tail losses are catastrophic. 

- **DeepScalper (Sun et al, 2022):**  This framework addresses **intraday trading** with a specialized RL agent.  Key ideas include a *dueling Q-network with action branching* (to handle large intraday action spaces like “go flat, hold position, add position” concurrently) and a *multi-modality encoder-decoder* that fuses macro and micro market inputs.  Crucially, DeepScalper adds a *risk-aware auxiliary task* to its loss: the agent is explicitly rewarded for balancing profit vs drawdown.  On 3 years of 6 futures data, DeepScalper outperformed baselines on multiple profit and risk metrics.  This shows that carefully designed RL (especially with risk auxiliary tasks) can capture fleeting intraday edges.  We could take inspiration by: (1) building an RL agent state with both high-level (overnight, session move) and low-level (minute-by-minute, orderflow) features, and (2) incorporating drawdown or CVaR penalties in the RL reward.

- **Risk-Aware RL in Portfolios:**  Recent work by Lwele et al. (2025) applies PPO for *dynamic portfolio allocation* with risk controls.  They use a *Sharpe-ratio-based reward* and impose explicit max-drawdown and volatility constraints in training.  This yielded adaptive, robust allocations under stress.  While our setting is single-contract trading, the same idea applies: penalize drawdowns in the objective (e.g. add $–\alpha \times \max{\rm DD}$ to reward) or limit volatility exposure. 

### Supervised/Time-Series ML

- **Neural Sequence Models:**  Recurrent and Transformer models excel at sequential data.  An “enhanced Transformer” has been applied to quant trading: Zhang & Chen (2024) show that, by capturing long-term dependencies and even leveraging sentiment embeddings, a Transformer factor model outperformed 100 conventional stock-selection factors and yielded robust signals.  Transformers naturally encode entire recent price history and can handle variable input (e.g. adding external features like COT, yield curves, or ICT indicators).  Their **benefits** include learning from longer context and non-linear patterns; **drawbacks** are data hunger and complexity.  In futures day-trading, a transformer could potentially integrate 1-min price history, higher-timeframe trend, and numeric indicators into a joint prediction of near-term move or optimal exit.

- **Diffusion Models:**  Very recent work introduces diffusion (score-based) models for finance.  Wang & Ventre (2024) repurpose a denoising diffusion model to *clean price series*.  They show that denoising noisy time-series improves return prediction: downstream classifiers had higher accuracy, and trading on the denoised data (e.g. MACD signals) produced **higher profits with fewer trades**.  Diffusion models can thus serve as a powerful feature extractor or data augmentation tool.  For our bot, a diffusion denoiser could be applied to the 1-min (or finer) bars to filter out micro-noise before signal generation.  This may increase SNR in features like momentum or volume imbalance.  Alternatively, diffusion could generate realistic synthetic trajectories to stress-test the strategy.

- **Graph and Hybrid Models:**  There is growing interest in Graph Neural Networks (GNNs) for market data, e.g. modeling cross-asset relationships or orderbook topology.  For futures, one could model the *orderbook as a graph* (levels as nodes connected by price steps) to predict likely next changes.  Hybrid models that combine LOB (orderbook) inputs with price history (a dual CNN/RNN) have shown promise in forecasting mid-price.  We should consider a multi-input architecture: e.g. a CNN on a crafted “image” of recent bid/ask stacks plus a sequential net for prices.

- **Feature Engineering:**  Literature emphasizes that **quality of state representation is critical**.  A study on RL for forex (TU Delft, 2024) found that *more indicators and longer history often hurt performance*.  In fact, “a single momentum or volatility indicator outperformed combinations” because extra features introduced noise.  The takeaway is to keep features parsimonious: include only those with proven signal.  On the other hand, adding agent state (current position, elapsed time) helped their agent.  For our case, useful features include: **current P/L**, **time-in-trade**, and **market context** (e.g. session time, overnight gap).  We must avoid high-dimensional bloated input or an RL agent may simply memorize past sequences.  Normalization is also key: all ML models should use log-returns or z-scores rather than raw price to stabilize training.

- **Microstructure Features:**  Academic work on limit-order books (LOB) suggests that *microstructural properties affect predictability*.  Briola et al. (2024) show that assets with different liquidity/volume profiles yield different forecast accuracies, and high accuracy does **not** always translate into profitable signals.  They advocate measuring “practical prediction quality” by the *chance of forecasting full transactions* rather than just mid-price changes.  For us, this means we should focus on features that relate to **execution probability**: e.g. bid-ask spread, orderbook imbalance, tick-by-tick momentum.  Especially in futures, looking at the top-of-book spread and change in queue volume can hint at order flow pressure.  We should compute features like *buy/sell volume imbalance*, *spread width*, *time since last print*, etc.  Additionally, industry “Smart Money” indicators (FVG, order blocks, liquidity levels) may encode points where big orders lurk, which ML can use as signals or state constraints.

### Strategy Ensembles and Meta-Controllers

- **Multiple Strategies:**  Rather than a single rule, combining strategies can hedge regime risk.  For example, overlay a momentum breakout model with a mean-reversion or “sweep-reclaim” model.  We already have `s1_sweep_fvg` and `s2_reclaim`; data shows they individually lost (PF<1) but perhaps only in certain regimes.  A meta-controller (possibly RL) could learn to *switch* between modes: if volatility is rising, favor breakout; if rangebound, favor mean-revert.  Recent work uses “multi-agent RL” for portfolios where each agent focuses on one style, and a gating mechanism allocates capital.  We could prototype a simple ensemble (e.g. trade only when all signals agree, or weight them by predicted confidence). 

- **Hierarchical RL:**  We can treat decision-making as hierarchical: a high-level RL agent selects or scales strategy sub-agents based on market state.  For instance, one RL policy decides position sizing or strategy weight (based on volatility, time-of-day, skew) and underlying fixed rules generate actual trades.  This might capture why a simple $200 kill is too coarse: an RL controller could reduce size or tighten stop if the portfolio drawdown or market volatility passes a threshold.

### Backtest Realism and Evaluation

- **Realistic Execution:**  Past tests already include NinjaTrader fill realism (2-tick adverse fills, etc).  We must maintain this rigor.  Any ML-filtered strategy should be tested with exactly the same slippage model.  For evaluation, walk-forward backtests and bootstrap Monte Carlo (as already done) are critical.  We will continue to use pass30/pass60, *maximum drawdown*, *profit factor*, *Sharpe/Sortino* of end-of-period P&L, and Monte Carlo “bust rate” as key metrics.  For ML models, classical metrics (AUC, precision at top decile) guide model selection.  

- **MFE/MAE Labeling:**  If we predict trade outcomes (e.g. exit profit/loss), consider using MFE/MAE labeling to capture hidden profit potential.  Labels like “did this planned trade eventually profit >x%” or regression of best-case profit could help train the ML models on more than binary win/lose. 

- **Survival Analysis for Stops:**  The idea of using survival models for stop-loss design is novel.  One could analyze historical trades using survival regression to predict the *hazard* of hitting a stop vs profit target, based on factors.  (Richards et al. used survival analysis on retail trades).  We may apply this to our data to estimate how likely a trade is to reach stop given time-in-market and volatility, potentially setting adaptive stop distances.

## Public ICT/SMC Repositories (Feature Sources)

We reviewed several open repositories for Smart-Money (ICT) indicators and strategies.  A summary is in the table below.

| Repo (Link)                           | Stars | Languages    | Key Features/Modules                            | License   | Integration Effort | Comments                      |
|---------------------------------------|------:|--------------|-----------------------------------------------|----------|-------------------|-------------------------------|
| joshyattridge/smart-money-concepts | 1.7k | Python       | **Indicators:** Order Blocks, FVGs, Liquidity Zones, Fair Value Gaps, Sessions, Fibonacci, SMC filters | MIT      | Low (pip install)  | Mature, actively maintained. Python APIs to compute ICT/SMC patterns from OHLCV/BBO. Very useful for feature engineering. |
| manuelinfosec/profittown-sniper-smc   |   58 | Python       | **Strategy skeleton:** BOS/OB detection, liquidity sweeps, Fib levels, multi-timeframe scans, risk modules (fixed SL/TP) | _Unclear_ | Medium            | Example sniper bot applying OB-based entries. Good for reference on implementing ICT-style rules, but not production-ready. |
| Ju571nK/ChartNagari                  |   11 | Go + React   | **Signals:** Multi-timeframe ICT/Wyckoff signals, Market Profile, ICT FVG/OB detection, Alerts | MIT (likely) | High (rewriting/adapting) | Offers a full charting/signals app. The logic (in Go) could inspire pattern detection, but integration would require porting algorithms to Python. |
| NadirAliOfficial/STAR-EA-v11.20     |    9 | MQL5 (MT5)   | **EA:** 12-scenario market classification, OB/FVG/OTE detection, liquidity/judas swing, segmented strategy rules  | Unknown (likely GPL) | High (reimplementation) | Complex ICT-based Expert Advisor. Could inform strategy variants (e.g. OTE entries), but code is in MQL5. Non-Python. |
| _Others (e.g. Pine libraries)_       |  –   | –            | –                                             | –        | –                 | Many SMC scripts exist, but closed environment. |
  
Each of the above could contribute modules or ideas.  In particular, joshyattridge’s Python library is ready-made for computing ICT patterns (we should leverage it for FVG, order-block, liquidity zone features).  The smaller repos provide tactical inspiration: e.g. Profittown’s OB filter logic or STAR EA’s “scenario engine” (classifying market states).  Integration effort ranges from *easy* (importing Python SMC code) to *significant* (rewriting Go/MQL logic).  None of these repos has large creditable backtesting, so any adopted component must be validated against our data.

## Data Acquisition and Feature Engineering

To enable advanced ML models and microstructure features, we recommend the following data enhancements (prioritized by cost-benefit):

| Data Type                     | Description                             | Frequency/Depth        | Cost Estimate (***=high***) | Benefit                                                         |
|-------------------------------|-----------------------------------------|------------------------|----------------|----------------------------------------------------------------|
| **Trade Ticks**               | Raw trade prints (price, size, side)    | Tick-by-tick (all trades) | **\*** (modest)   | Enables volume and orderflow features (imbalance, footprint). Improves accuracy of executed-price modeling. |
| **Top-of-Book Quotes (BBO)**  | Best bid/ask quotes with sizes         | Tick-level quotes       | **\** (moderate)  | Captures real-time liquidity, spread, imbalance. Essential for modeling slippage and queue pressure. |
| **Multi-Level LOB Depth**     | Order-book depths (e.g. top 5 levels)   | Tick-level             | *** (expensive) | Full depth data allows precise prediction of fill probabilities, but high cost. Consider sampling only major regime days if needed. |
| **1-sec Bars**                | Aggregated OHLCV per second             | 1-second bars          | \* (low)      | Granular data for faster feature computation (e.g. ATR, momentum). Good fallback if tick data unavailable. |
| **Macro/News Calendar**       | Scheduled event times (FOMC, CPI, etc)  | Timestamped events     | Free           | Avoid trading or tighten stops around major announcements. Enables volatility regime classification. |
| **Related Market Data**       | E.g. interest rates, other indices      | 1-min or tick         | \* (free/low)  | For example, monitor bond yields or VIX futures intraday as volatility indicators. (Optional) |
| **Options Implied Vol (VIX)** | Short-term implied volatility index     | 1-min/update         | \* (free via API) | Can signal market fear; use as filter or dynamic sizing factor. |
| **Horiz. Expiry Data**        | Multi-month futures (ES, NQ multi-series) | 1-min bars             | \* (free)      | Model term structure effects if multiple contracts trade concurrently. |
  
*Cost Estimation:*  **\*** = ~$500–1000, **\** = ~$200–500, *\* = <~$100 (per symbol for 2 years, vendor dependent). Tick/BBO data can be obtained from futures data vendors (e.g. TickData, AlgoSeek, Neon).  If budgets are very tight, 1-second bars and top-of-book quotes are the highest priority (they capture most liquidity info at low cost).  Full depth is optional – a pilot purchase for 1 month could gauge benefit.  Macro calendars are mostly free (e.g. Econoday API).

*Feature Ideas:*  Using these data, we can compute:  

- **Order Flow Imbalance:** ratio of buy-volume/(buy+sell) over short window; often predictive of short-term moves.  
- **Limit Order Book Features:** spread, depth difference, liquidity sweep detection (sudden dip in one side volume).  
- **Price/Momentum Indicators:** ATR-based volatility, Momentum (price differential), VWAP deviations.  
- **SMC Indicators:** FVG (fair value gaps), Order Blocks (recent swing pivots), Bos/Liquidity pools via the Python SMC library.  
- **Agent State:** current P/L, time-in-trade, contracts held (for RL).  

All features should be **normalized** (e.g. z-score over rolling window) as best practice.  We will also include multi-timeframe labels: e.g. the trend on 5-min vs 1-min to filter only co-trending conditions.

## Candidate ML Architectures

We consider a range of ML models.  A comparative table is below:

| Model Type              | Key Features                                        | Pros                                               | Cons                                                |
|-------------------------|-----------------------------------------------------|----------------------------------------------------|-----------------------------------------------------|
| **Gradient Boosted Trees (XGB/LightGBM)** | Ensemble of decision trees on tabular features | Strong baseline, handles heterogeneous features, fast train, interpretable importance | Struggles with very long sequences or raw time data, less effective with highly non-linear temporal patterns. |
| **Feed-Forward Neural Net (MLP)** | Dense network on engineered features            | Can approximate complex non-linear relationships, good with feature-rich input (our MLP-lite gave AUC ~0.75) | Requires careful tuning; may overfit if too large; fixed input length. |
| **Recurrent NN (LSTM/GRU)** | Sequence model capturing temporal dynamics      | Captures sequential dependencies; suitable if trade outcome depends on price path | Harder to train, vanishing gradients for long sequences, slower inference. |
| **Temporal CNN**        | 1D convolutions over time series                  | Learns local temporal patterns (e.g. price motifs), efficient training | Limited receptive field (unless stacked) for long-range features; needs uniform time steps. |
| **Transformer (Seq2Seq)** | Multi-head attention for time series             | Captures long-range dependencies and cross-feature interactions, handles missing data via masks | Very data-intensive, heavy compute, risk of overfitting on small datasets. |
| **Reinforcement Learning (DQN, PPO, SAC)** | Learns policy (action=trade) from interaction   | Can optimize directly for P&L with risk signals; adaptive | Requires careful reward design, large data (simulated), sensitive to non-stationarity; training complexity. |
| **Diffusion/Score-Based Model** | Denoising or generative approach               | Can improve signal quality (denoising) or simulate realistic paths; novel research shows boosts classification | Complex to implement; current applications are research-stage; heavy compute. |
| **Graph Neural Network (GNN)** | Models relations (e.g. levels of orderbook, or cross-market graph) | Captures structured dependencies (multi-level LOB as graph) | Experimental in trading; requires careful graph design; data-hungry. |

Each model will be evaluated on our data with target metrics (see “Evaluation Metrics” below).  Tree models and MLPs serve as fast baselines.  Transformers and RNNs may extract more nuance but risk overfitting given limited data (2 years of 1-min data ≈ few 10<sup>5</sup> points).  Reinforcement learning could be applied for a high-level agent (e.g. position sizing), but not as a first step due to complexity.  We will start with supervised models and simple RL.

## Proposed Pipeline (Flowchart)

```mermaid
flowchart LR
    A[Raw Data (price ticks, BBO, news/calendar)] --> B[Feature Engineering<br/>(1s bars, indicators, SMC patterns)]
    B --> C[Machine Learning Models]
    C --> D[Predicted Signals or Regime Probabilities]
    D --> E[Strategy Logic]
    E --> F[Risk Management]
    F --> G[Trade Execution & Portfolio Monitoring]
    G --> H[Backtesting & Monte Carlo Validation]
    H --> I[Metrics (pass30, PF, Sharpe, MaxDD)] 
    I --> L[Iterate/Refine]
```

*(Mermaid diagram: Data → Features → Models → Signals → Strategy + Risk → Execution → Evaluation → Iterate.)*

## Implementation Roadmap & Experiments

We outline a staged plan, with ablation tests to quantify each change:

| Step | Task | Data/Tools | Duration | Metrics/Notes |
|------|------|-----------|----------|---------------|
| 1 | **Data Expansion:** Acquire 1s trades + BBO data; incorporate macro calendar events. | Vendor feeds, APIs | 1–2 weeks | Verify data quality (no gaps). |
| 2 | **Feature Pipeline:** Implement new features: ATR, spreads, volume imbalance, SMC indicators (FVG/OB via [1]), time-of-day, news flags. | Python (pandas, SMC lib) | 1 week | Correlation analysis, feature importance (e.g. via trees). |
| 3 | **Baseline Modeling:** Train models on current labeling (e.g. Win/Loss or return quantiles) using **XGBoost and MLP** on new features. | Scikit-learn, PyTorch | 2 weeks | Evaluate AUC, precision@top10%. Check gains over base metrics. |
| 4 | **Backtest with Signal Filter:** Integrate top model as a filter (e.g. skip trades predicted to lose). | Backtester + Python API | 1 week | Compare pass30/pass60, PF vs baseline. Ablation: with/without model filter. |
| 5 | **Deep Learning Model:** If data permits, try an RNN or Transformer using sequences of recent bars (and maybe news). | TensorFlow/PyTorch | 2 weeks | Compare to MLP: does it improve classification or reduce large loss trades? Validate via walk-forward. |
| 6 | **Stop-Loss Optimization:** Test dynamic stop rules (e.g. ATR-based stops, scaled stops by volatility). Possibly use survival analysis to set level. | Python (statsmodels) | 1 week | Measure reduction in maxDD and effect on return. Adjust to maintain expectancy. |
| 7 | **Ensemble/Meta Strategy:** Combine existing strategies (e.g. overlay S1/S2 rules) with voting or weighted signals from ML. | Backtest scripts | 1 week | Test ensemble PF and win-rate. See if combined pass30 improves. |
| 8 | **Risk-aware RL Prototype:** Formulate a simple RL environment: state includes current P/L and volatility; actions adjust position size or stops. Use PPO with CVaR reward. | RL toolkit (Stable-Baselines) | 3 weeks | Compare average return and drawdown vs fixed strategy. (High risk of failure -> small sim). |
| 9 | **Monte Carlo Validation:** For each candidate config, run 5k+ MC paths to estimate pass30, bust rates (as in [monte_carlo_1k]). | Python (numpy/pandas) | 1 week | Summarize results in table (like control vs new). Ensure no increase in bust beyond acceptable. |
| 10 | **Paper Trading Simulation:** If possible, run on live paper account (simulated fills) for a short period to check real-time viability. | Trading platform simulator | 2+ weeks | Real-time slippage analysis, check if real fill model holds. |

Each ablation (e.g. adding SMC features only, or risk changes only) should be tested independently to measure impact on pass30, PF, and drawdowns. We prioritize *safety*: any change should not degrade out-of-sample worst-case significantly.  

## Evaluation Metrics

- **FundedNext Criteria:** Primary: % of MC simulations that pass 30-day goal (target ~60%+). Also track 60-day pass. Require bust≈0.  
- **Profitability:** Expectancy per trade, Profit Factor (PF), Annualized Sharpe/Sortino.  
- **Risk Metrics:** Maximum Drawdown, 95% CVaR, Kelly fraction. (We will compute CVaR from MC).  
- **Hit-Rate (WR):** Keep above ~60%.  
- **AUC / Precision:** For any ML model, track AUC and precision of top predictions. Check stability on rolling windows.  
- **Sample Efficiency:** ML models’ training time & compute (TFLOPS), to ensure feasibility for retraining.  
- **Implementation Complexity:** Maintain code coverage tests; each new module must be covered by unit tests to prevent regressions.  

## Integration Checklist

Before moving to live run, ensure the following:

- [ ] **Data Pipeline:** New data sources ingested and aligned (time-sync 1s trades/BBO with 1m bars).  
- [ ] **Indicator Validation:** Verify SMC indicators (FVG/OB) produce expected patterns on known chart segments.  
- [ ] **Model Connect:** Embed ML model prediction into strategy engine without altering existing logic except for gating.  
- [ ] **Risk Module:** Implement new stop-sizing or kill-switch logic and validate through test scenarios (simulate a loss >$X triggers kill).  
- [ ] **Config Flags:** Allow toggling experiments (e.g. enable_features=true, dynamic_stops=true) to compare easily.  
- [ ] **Backtest Consistency:** All candidate strategies (old vs new) run on identical data with same slippage.  
- [ ] **Documentation:** Update strategy documentation with new parameters and logic.  
- [ ] **Paper Run Readiness:** Dry-run in real-time simulation (no execution) for at least 1 funded challenge length.  

All code changes should be version-controlled with extensive tests.  Once research-phase validations show clear benefit, we can merge into the main trading code for a final backtested “paper” evaluation.

## References

We draw on current literature in quantitative trading and ML.  Notable sources include risk-aware RL frameworks, distributional RL for CVaR, and diffusion-based time-series denoising for finance.  Feature engineering insights come from recent RL trading studies, and LOB forecasting research highlights microstructure limits on predictability.  

