# Deep Analysis & Improvement Roadmap for FN25K Frontier

## 1. Diagnostic Summary: Where You Actually Are

Your bot has undergone a remarkable trajectory:

| Era | pass10 | pass20 | median | PF | avg loss | Core mechanism |
|---|---|---|---|---|---|---|
| MBO quality base | 9.6% | 51.9% | 20d | 11.62 | -$53.57 | Toxic-flow skip, low density |
| Cap expansion | 28.8% | 80.8% | 15d | 3.11 | -$48.07 | score_20 add-on ranker, 3.16 t/day |
| **Mode allocator (current)** | **46.2%** | **98.1%** | **11.5d** | **3.22** | **-$42.62** | **Regime-conditional mode selection** |

You've essentially traded quality for speed and it worked. But the frontier is now constrained by three structural problems that no amount of parameter tuning within the current architecture will solve:

**Problem A — February Early Accumulation**: Feb pass10 = 10%, day10_total_median = $590 vs $1,225 (Jan). The allocator fixes Feb pass20 (100%) but can't accelerate early PnL because its constituent modes all scratch too aggressively in choppy conditions, and the "fast" mode only works when shadow state is already green.

**Problem B — Second-Half Degradation**: First-half pass10 = 53.8%, second-half = 38.5%. The allocator's fixed thresholds (especially `m_lb3_nq_es_disp_mean <= 0.0018581831...`) are overfitted to the full-sample distribution. As the market regime drifts, branch assignment degrades.

**Problem C — Exit Efficiency**: Avg win $48.73 on ~3.38 trades/day means you're capturing a fraction of available favorable excursion. The scratch mechanism limits losses but also caps upside in trending conditions. This is the structural cost of the speed strategy.

Below are **concrete, prioritized improvements** organized by expected impact and implementation complexity.

---

## 2. HIGH-IMPACT IMPROVEMENTS

### 2.1 Intra-Challenge Dynamic Mode Switching (Expected: +10-15% Feb pass10)

**Current limitation**: The allocator picks ONE mode at challenge start and holds it for the entire challenge. This is a static open-loop decision.

**Proposed change**: Allow mid-challenge mode transitions based on causal cumulative state:

```python
def select_mode_dynamic(day, cum_pnl, daily_pnl_history, regime_features):
    # Phase 1: Days 1-5 — "Acceleration Window"
    if day <= 5:
        if regime_features['nq_es_disp'] <= adaptive_disp_threshold():
            # Clean regime: use runner with extended targets
            return 'runner_extended_target'
        elif cum_pnl < -50:
            # Early deficit: switch to conservative scratch + larger confirmed entries
            return 'conservative_scratch_plus_confirmed'
        else:
            return current_allocator_branch(regime_features)
    
    # Phase 2: Days 6-10 — "Catch-Up Window"  
    if day <= 10:
        gap_to_target = 1250 - cum_pnl
        days_remaining_urgent = 10  # still want pass20
        required_pace = gap_to_target / days_remaining_urgent
        
        if required_pace > 80:  # behind pace
            # Only take high-conviction trades, size up slightly
            return 'high_conviction_sized'
        elif required_pace < 40:  # ahead of pace
            return 'defensive_lock'  # protect gains
    
    # Phase 3: Days 11+ — current allocator logic
    return current_allocator_branch(regime_features)
```

**Why this helps February**: Feb starts slowly because the allocator commits to a mode that isn't optimal for the first 5-10 days of that particular challenge. By making the mode decision state-dependent rather than start-dependent, you can adapt when the initial mode isn't accumulating fast enough.

**Validation approach**: Replay all 52 starts with dynamic switching. Key check: does second-half pass10 improve? Does Feb pass10 move above 20% without damaging pass20 or max DD?

**Implementation complexity**: Medium. Requires re-running the allocator lab with a state-dependent policy function.

---

### 2.2 Adaptive Thresholds via Rolling Quantiles (Expected: +5-8% second-half pass10)

**Current limitation**: The allocator uses the threshold `m_lb3_nq_es_disp_mean <= 0.0018581831...` which is suspiciously precise — it's clearly optimized on the full sample. The split-half degradation (53.8% → 38.5%) confirms this is overfitting.

**Proposed change**: Replace all fixed allocator thresholds with rolling-window quantiles:

```python
def adaptive_disp_threshold(lookback_days=60, quantile=0.4):
    """Use 40th percentile of prior 60 trading days' dispersion as the boundary"""
    historical_disp = get_dispersion_series(lookback_days=lookback_days)
    return historical_disp.quantile(quantile)

def adaptive_shadow_threshold(lookback_starts=14, quantile=0.6):
    """Shadow quality boundary adapts to recent shadow performance distribution"""
    historical_shadow_sums = get_shadow_sum_series(lookback_starts=lookback_starts)
    return historical_shadow_sums.quantile(quantile)
```

**Why this helps**: The market's dispersion distribution shifts over time. A fixed threshold that was the 40th percentile in Dec-Jan might be the 60th percentile in Feb-Mar, causing wrong branch assignments. Rolling quantiles maintain the same percentile rank regardless of distribution shift.

**Critical detail**: The lookback must be causal — only use data from BEFORE the challenge start. Use expanding windows for early starts (first 20 starts) and rolling windows after that.

**Validation**: Re-run split-half and monthly sensitivity. The gap between first-half and second-half should narrow. The 10k bootstrap pass10_mean should increase.

---

### 2.3 MFE-Aware Adaptive Exit System (Expected: +$5-8 avg win, −0.1 trades/day)

**Current limitation**: All modes use fixed target/scratch exits. The avg win ($48.73) vs avg loss ($42.62) gives R:R of 1.14 — barely positive. Meanwhile, the MBO quality base had R:R of 1.26 ($67.50/$53.57), meaning the speed modes are leaving significant favorable excursion on the table.

**Proposed change**: Implement a two-tier exit system:

```
Tier 1: Base exit (current target/scratch)
Tier 2: MFE-based trailing exit
  - When trade reaches 1.5x initial target, switch to trailing stop
  - Trail at 40% of MFE (i.e., give back no more than 60% of peak profit)
  - Maximum hold time: 45 minutes (prevent capital tie-up)
```

**Implementation approach**:
1. Analyze MFE distribution for all winning trades across all modes
2. Identify what fraction of MFE is captured by current fixed targets
3. Build a simple rule: if price reaches `entry + 1.5 * target_distance`, switch to trailing stop at `peak_price - 0.4 * (peak_price - entry)`
4. Cap the trailing at 45 minutes to prevent zombie trades

**Why this helps February**: In adverse regimes, when you DO get a winner, you need to extract maximum value from it. Currently, the fast scratch mode cuts winners at relatively modest targets. Letting winners run in confirmed trend moves (which DO occur even in February) could push avg win from $48 to $55+ without changing the entry logic at all.

**Validation**: This changes the replay engine, not just the allocator. Need to re-run all stress/bootstrap/monthly checks. Key metric: does avg win increase without increasing avg loss or reducing win rate significantly?

---

### 2.4 Account-Level Sequence Model (Expected: +3-5% overall pass10, better Feb)

**Current limitation**: The score_20 model predicts trade-level outcome probability. But the real objective is account-level: "Will taking this trade help me reach $1,250 within 30 days given my current account state?" These are NOT the same thing. A 60%-probability trade might be the right decision if you're at $1,100 on day 25, but the wrong decision if you're at $200 on day 3 with $180 daily loss remaining.

**Proposed change**: Train a sequence model that predicts challenge completion probability given state-action pairs:

```python
# State features (all causal)
state_features = [
    'day_in_challenge',           # 1-30
    'cumulative_pnl',             # current account PnL
    'pnl_to_target',              # 1250 - cum_pnl
    'days_remaining_to_20',       # max(0, 20 - day)
    'days_remaining_to_30',       # max(0, 30 - day)
    'trailing_dd_buffer',         # distance to EOD trailing DD
    'daily_pnl_so_far',          # today's realized PnL
    'daily_loss_remaining',       # 200 + daily_pnl_so_far (if negative)
    'num_trades_today',           # trade count today
    'recent_3d_win_rate',         # WR over last 3 trading days
    'recent_3d_pnl_sum',          # PnL over last 3 trading days
    'nq_es_disp_current',        # current regime dispersion
    'nq_es_return_today',        # leader flow today
]

# Action features (for the candidate trade)
action_features = [
    'score_20',                   # current model probability
    'predicted_ev',               # expected value
    'entry_signal_strength',      # signal confidence
    'proposed_risk_per_contract', # stop distance
]

# Target: did challenges that took similar trades in similar states pass?
# This is a survival analysis / sequence prediction problem
```

**Model architecture**: Use a **Decision Transformer** or **Trajectory Transformer** approach:
- Input: sequence of (state, action, reward) tuples from past challenge trajectories
- Output: probability of reaching $1,250 within N remaining days
- Train on ALL challenge trajectories (both passing and failing starts)
- The model learns WHEN to be aggressive vs conservative based on account state

**Training data construction**: From your 52+ replay starts, extract every decision point (trade opportunity) with:
- State at decision time
- Whether the trade was taken or skipped
- What happened to the challenge after that decision

This gives you ~52 starts × ~17 trades × multiple decision points = ~1,500+ training examples. Not huge, but enough for a focused model.

**Why this is better than score_20_v2**: score_20_v2 tried to improve trade-level prediction. The account-level model directly optimizes the actual objective. A trade with score_20 = 0.15 might be worth taking if you're at $1,150 on day 12, but not if you're at $100 on day 2. The current system can't distinguish these cases.

**Implementation complexity**: High. This is a new model training pipeline. Start with a simple logistic regression on state-action features before moving to sequence models.

---

## 3. MEDIUM-IMPACT IMPROVEMENTS

### 3.1 Volatility-Regime Conditional Scratch Thresholds

**Observation**: The current scratch modes use fixed dollar thresholds. But a $20 scratch means different things in different volatility environments:
- In low-vol: $20 might be 2 ATR — too tight, scratching trades that would work
- In high-vol: $20 might be 0.5 ATR — too loose, letting losers run too far

**Proposed change**: Scale scratch thresholds to ATR:

```python
def adaptive_scratch_threshold(instrument='MNQ'):
    atr_5min = compute_ATR(instrument, period=20, bar_size='5min')
    base_scratch_atr_multiple = 0.8  # calibrate from MFE/MAE analysis
    return max(base_scratch_atr_multiple * atr_5min, 15)  # floor at $15
```

**Expected impact**: In February's potentially different volatility profile, this could let winners breathe more while still cutting losers promptly. Might improve Feb pass10 by 5-10 percentage points.

### 3.2 Cross-Session Gap Analysis for Daily Start Decision

**Observation**: The allocator uses prior 3-day NQ/ES features but doesn't specifically model the overnight/session gap behavior, which is often the strongest predictor of intraday regime.

**Proposed change**: Add gap features:
- `overnight_gap_pct`: (RTH open - previous RTH close) / previous RTH close
- `gap_fill_probability`: probability that the gap will be filled today (based on historical gap statistics by size and direction)
- `pre_market_trend`: 30-min pre-market NQ/ES trend direction and strength

Use these to adjust the day's mode selection at RTH open, not just at challenge start.

### 3.3 Ensemble Mode Agreement Signal

**Current**: The allocator picks one mode. When modes disagree, you're fully committed to whichever one was selected.

**Proposed change**: Run all four modes in shadow for the first 3 days of each challenge. Count signal agreement:
- If ≥3 of 4 modes agree on a trade: full position size
- If 2 of 4 agree: half position size
- If only 1 signals: skip

After day 3, lock in the mode with the best shadow performance for that specific start.

**Expected impact**: Reduces the risk of choosing the wrong mode at challenge start. May slow day-1-3 slightly but could improve pass10 by preventing bad mode assignments.

### 3.4 Daily Loss Budget Allocation by Challenge Phase

**Current**: Fixed $200 daily loss kill regardless of challenge state.

**Proposed change**: Phase-dependent daily loss budget:

| Phase | Days | Daily Loss Budget | Rationale |
|---|---|---|---|
| Early acceleration | 1-5 | $150 | Preserve capital, build cushion |
| Accumulation | 6-15 | $200 | Standard risk |
| Protected lead | 16+ (if PnL > $800) | $150 | Lock in gains, avoid giving back |
| Catch-up | 16+ (if PnL < $600) | $175 | Moderate aggression, can't afford big loss |

**Expected impact**: Small but consistent improvement in floor gap and max DD. The tighter early budget prevents the worst-case scenario: a big day-1 loss that puts you in catch-up mode for the entire challenge.

### 3.5 Pre-Computed Daily MBO Summary Features

**Observation**: Report 13/14 showed that real-time MBO book reconstruction was too slow in Python, and fine-window MBO skips hurt pass speed. But MBO information IS valuable — the toxic-flow skip was the original quality breakthrough.

**Proposed change**: Instead of real-time MBO processing, compute daily MBO summary statistics as offline features:
- `prev_day_mbo_signed_flow`: total signed order flow from previous day
- `prev_day_mbo_add_cancel_ratio`: ratio of adds to cancels (high = real interest, low = spoofing)
- `prev_day_mbo_trade_imbalance`: net aggressive buying vs selling

These are computed once per day (offline) and used as regime features for the NEXT day's allocator decisions. No real-time processing needed.

**Expected impact**: Could improve regime detection accuracy, particularly for identifying when MBO flow supports vs contradicts the signal direction. Might improve Feb by detecting adverse flow conditions earlier.

---

## 4. RESEARCH DIRECTIONS (Longer-Term, Higher Risk)

### 4.1 Reinforcement Learning for Exit Timing

Train an RL agent (PPO or SAC) specifically for the exit decision:

- **State**: current PnL, time in trade, recent price action features, book imbalance (if available)
- **Actions**: {hold, scratch_exit, partial_profit_50%, full_profit_exit}
- **Reward**: trade PnL minus opportunity cost (measured by what the next-best trade would have earned)
- **Training**: Use the trade-level data from all modes, treating each trade as an episode

This could discover exit policies that no fixed rule can express — for example, "hold through a $15 adverse excursion if the MBO flow is strongly supportive" or "scratch immediately at +$5 if MBO flow reverses."

### 4.2 Generative Data Augmentation for February

The fundamental problem with February is that you only have 10 starts. This is insufficient to learn February-specific mode selection.

**Approach**: 
1. Identify the February regime characteristics from NQ/ES features
2. Use a conditional GAN or diffusion model trained on normal-regime trade data, conditioned on February-like features
3. Generate synthetic trade sequences that mimic February behavior
4. Augment the allocator training with synthetic February starts
5. Validate on the real 10 February starts only (never train on them)

This is ambitious but could unlock February-specific adaptation without overfitting to 10 data points.

### 4.3 Causal Inference for Feature Selection

**Why score_20_v2 failed despite better AUC**: The model likely picked up features that correlate with good outcomes but don't cause them. For example, "low dispersion at entry time" might correlate with winning trades, but it's actually the CAUSE of the low dispersion (institutional flow absorbing liquidity before a move) that matters, not the dispersion itself.

**Approach**:
1. Apply Granger causality tests to all features vs trade outcomes
2. Use do-calculus to identify features where intervening changes the outcome distribution
3. Remove features that are effects (not causes) of trade outcomes — these are look-ahead-adjacent
4. Retrain score_20 on the causally-selected feature set

### 4.4 Multi-Objective Bayesian Optimization for Parameter Search

Your current grid search over threshold/risk/cap is exhaustive but coarse. Bayesian optimization with multi-objective acquisition (e.g., Expected Hypervolume Improvement) can:
- Find Pareto-optimal parameter combinations more efficiently
- Discover non-obvious interactions (e.g., threshold=0.18 works well only with risk=0.28 and cap=4)
- Quantify the tradeoff frontier between pass10 and max DD

---

## 5. IMPLEMENTATION PRIORITY

| Priority | Improvement | Expected Impact | Complexity | Dependencies |
|---|---|---|---|---|
| **1** | Adaptive thresholds (rolling quantiles) | +5-8% pass10, fixes second-half degradation | Low | None — change allocator thresholds only |
| **2** | Intra-challenge dynamic mode switching | +10-15% Feb pass10 | Medium | Needs replay engine modification |
| **3** | MFE-aware adaptive exits | +$5-8 avg win | Medium | Needs MFE analysis, replay engine change |
| **4** | Account-level sequence model | +3-5% overall pass10 | High | New model training pipeline |
| **5** | Volatility-scaled scratch thresholds | +5-10% Feb pass10 | Medium | ATR computation in replay |
| **6** | Daily loss budget by phase | Small DD/floor-gap improvement | Low | Simple parameter change |
| **7** | Pre-computed daily MBO features | Better regime detection | Medium | MBO data pipeline |
| **8** | Ensemble mode agreement | Reduced mode-selection risk | Medium | Multi-mode shadow replay |
| 9 | RL exit agent | Unknown, potentially large | Very High | Full RL training infrastructure |
| 10 | Causal feature selection | Better model generalization | High | Causal inference expertise |

### Recommended Next Sprint (2-3 weeks):

1. **Implement adaptive thresholds** (2-3 days): Replace all fixed allocator thresholds with rolling quantiles. Re-run the full allocator lab. This should be a pure improvement with no downside risk.

2. **Build the dynamic mode switching replay** (5-7 days): Modify the replay engine to allow mid-challenge mode transitions based on cumulative state. Test the three-phase policy (acceleration / catch-up / lock-in).

3. **MFE analysis of current trades** (2-3 days): Compute MFE/MAE for all trades across all modes. Determine what fraction of MFE is captured by current exits. Design the adaptive exit system based on the data.

4. **Account-level model v0** (5-7 days): Start with a simple logistic regression predicting pass/fail from (state, action) features. This is the proof-of-concept for the sequence model approach.

The first two items alone could push overall pass10 above 55% and Feb pass10 above 25% without any tradeoff in pass20, max DD, or bust rate.

---

## 6. KEY INSIGHT: What's Really Limiting You

The deepest structural insight from these reports is this: **your bot has a precision-recall problem at the challenge level, not the trade level.**

Trade-level metrics are fine (74% WR, PF 3.22). The issue is that the challenge-completion objective is a sequential, path-dependent problem, and no amount of trade-level optimization perfectly maps to it. The allocator was a big step because it recognized that different STARTING CONDITIONS need different strategies. But it's still a one-shot decision at challenge start.

The next breakthrough will come from making the bot's decisions **state-dependent throughout the challenge**, not just at the start. This means:
- Knowing when you're behind pace and need to take calculated risks
- Knowing when you're ahead of pace and should lock in gains
- Adapting to intraday regime shifts, not just multi-day regimes
- Understanding that the "optimal" trade depends on the account state, not just the market state

The account-level sequence model (priority 4) is the ultimate expression of this idea, but even the simpler dynamic mode switching (priority 2) captures most of the benefit by recognizing that the right strategy changes as the challenge progresses.