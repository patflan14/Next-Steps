# Evolutionary Investment Framework: Complete System Design

**Date:** November 12, 2024
**Status:** Design Phase
**Goal:** Build a self-improving investment strategy system using evolutionary algorithms, LLM debates, and tournament selection

---

## Table of Contents

1. [Core Concept](#core-concept)
2. [System Architecture](#system-architecture)
3. [Evolution Framework](#evolution-framework)
4. [Debate Framework](#debate-framework)
5. [Meta-LLM & Tournament Selection](#meta-llm--tournament-selection)
6. [Allocation & Execution Strategy](#allocation--execution-strategy)
7. [Risk Management & Adaptation](#risk-management--adaptation)
8. [Critical Flaws & Mitigations](#critical-flaws--mitigations)
9. [Implementation Plan](#implementation-plan)
10. [Final Recommendations](#final-recommendations)

---

## Core Concept

### The Big Idea

Create an **evolutionary investment system** where:
1. **Initial "seed" prompt** describes an investment strategy
2. **Debate framework** refines the strategy through LLM argumentation
3. **Backtesting** validates historical performance
4. **Meta-LLM** synthesizes insights and creates variations
5. **Tournament selection** picks the best strategies
6. **Evolution continues** - winners spawn new variations
7. **Natural selection** - only best strategies survive

### Why This Could Work

- **Automated exploration** of strategy variations you might not think of manually
- **Objective fitness function** - backtest metrics don't lie
- **Preserves lineage** - trace what changes improved performance
- **Compound learning** - each generation builds on insights from previous
- **Parallel evolution** - multiple branches explore different directions

---

## System Architecture

### High-Level Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    GENERATION N                              │
└─────────────────────────────────────────────────────────────┘

For each strategy prompt:
    ↓
1. BACKTEST (test raw idea first)
    ↓
2. DEBATE FRAMEWORK
   - LLM1 ↔ LLM2 (multiple rounds)
   - Decreasing contentiousness
   - Logic ratings
   - Arguments only (no final strategy yet)
    ↓
3. META-LLM SYNTHESIS
   INPUTS:
   - Original prompt
   - Debate transcript + logic ratings
   - Backtest results
   - Current market data
   - News & sentiment
   - Historical correlations

   OUTPUTS:
   - ONE synthesized strategy (best from debates + context)
   - Confidence score
   - Forward projection
   - Rationale
    ↓
4. CREATE SYMPHONY & BACKTEST
    ↓
5. STORE RESULTS
   - Strategy details
   - Backtest metrics
   - Debate insights
   - Logic ratings
   - Confidence scores

Repeat for ALL strategies in generation N
    ↓
┌─────────────────────────────────────────────────────────────┐
│            META-LLM TOURNAMENT ANALYSIS                      │
└─────────────────────────────────────────────────────────────┘

INPUTS:
- All backtest results from Gen N
- All debate transcripts
- All logic ratings
- Current market data
- News/sentiment/correlations
- Historical lineage

TASKS:
1. Rank strategies by composite score:
   - Backtest performance (Sharpe, drawdown, returns)
   - Logic quality (debate ratings)
   - Forward projection (predicted future performance)
   - Market alignment (current conditions)

2. Select top N strategies (e.g., top 3-5)

3. Generate variations ONLY of top performers:
   - Create 2-4 variations per winner
   - Use debate insights for mutation ideas
   - Adapt to current market regime

OUTPUTS:
- Rankings of all Gen N strategies
- Top N selected strategies
- 6-20 NEW variation prompts (for Gen N+1)
- Forward projections for each
- Recommendations: which to keep, which to archive
    ↓
┌─────────────────────────────────────────────────────────────┐
│              GENERATION N+1 STARTS                           │
└─────────────────────────────────────────────────────────────┘
New variation prompts go through Phase 1 flow...
```

---

## Evolution Framework

### Parameters

**Recommended Configuration: V=4, G=6, K=40%**

- **V (Variations)**: 4 variations per strategy
- **G (Generations)**: 6 generations
- **K (Keep percentage)**: 40% (top performers advance)

### Why These Numbers?

```
Total Operations:
- 315 backtests
- 315 debates
- ~4.3 hours runtime
- 64 final strategies for tournament

Benefits:
- Deep enough for real refinement (6 generations)
- Enough exploration without noise (4 variations)
- Strong selection pressure (40% keeps quality high)
- Manageable daily execution time
```

### Evolution Mathematics

```
Generation n:
  Strategies entering: S_n
  After creating variations: S_n × (1 + V)
  After selection: S_(n+1) = floor(S_n × (1 + V) × K)
  Backtests/Debates: S_n × (1 + V)

Example (V=4, G=6, K=0.4):
  Gen 0: 1 seed → 5 total → keep 2
  Gen 1: 2 → 10 total → keep 4
  Gen 2: 4 → 20 total → keep 8
  Gen 3: 8 → 40 total → keep 16
  Gen 4: 16 → 80 total → keep 32
  Gen 5: 32 → 160 total → keep 64

  Total: 315 backtests, 64 final strategies
```

### Alternative Configurations

| Config | Backtests | Time | Final Strategies | Use Case |
|--------|-----------|------|------------------|----------|
| V=3, G=4, K=50% | 60 | ~50min | 16 | Quick daily |
| V=4, G=6, K=40% | 315 | ~4.3hr | 64 | **Recommended** |
| V=3, G=7, K=50% | 508 | ~7hr | 128 | Weekend deep dive |

---

## Debate Framework

### Purpose

**NOT to create final strategy** - debates generate arguments and explore edge cases.

**Purpose:**
- Evaluate logical soundness
- Identify assumptions
- Explore failure modes
- Generate alternative approaches

### Structure

```
LLM1 (Proponent):
  - Refines the strategy
  - Defends choices
  - Improves rationale
  - Temperature: 0.3

LLM2 (Critic):
  - Challenges assumptions
  - Proposes counterarguments
  - Identifies risks
  - Temperature: 1.0

Rounds: 3 debates
Contentiousness: Decreasing (1.0 → 0.7 → 0.4)

Each round:
  - LLM1 states position
  - LLM2 critiques
  - LLM1 responds
  - LLM3 rates logic (0-10)

OUTPUT:
  - Full debate transcript
  - Logic ratings from each round
  - Key arguments and counterarguments
  - NO final strategy (that's Meta-LLM's job)
```

### What Debates Should NOT Have Access To

**❌ NO backtest data**
- Forces focus on logical soundness, not curve-fitting
- Prevents hindsight bias
- Encourages exploration of edge cases

**❌ NO current market data**
- Evaluates timeless strategy logic
- Avoids recency bias
- Promotes regime-agnostic thinking

**✅ ONLY the strategy prompt**

### Separation of Concerns

```
DEBATE: Logical soundness
  - "Is this reasoning sound?"
  - "What assumptions does it make?"
  - "What could go wrong?"
  - "Are there better formulations?"

META-LLM: Performance + Context
  - "Did it work historically?"
  - "Will it work in current regime?"
  - "How does it compare to alternatives?"
  - "What's the forward projection?"
```

This separation creates **complementary perspectives**:
- Debate identifies logical flaws
- Meta-LLM validates with data
- Together: both logic AND evidence must support the strategy

---

## Meta-LLM & Tournament Selection

### Meta-LLM Role

**Phase 1: Strategy Synthesis (Per Strategy)**

```
INPUTS:
  - Original prompt
  - Debate transcript + logic ratings
  - Backtest results
  - Current market data
  - News/sentiment
  - Historical correlations

OUTPUT:
  - ONE synthesized strategy
  - Confidence score (0-100%)
  - Forward projection (expected return)
  - Rationale (why this synthesis)
```

**Phase 2: Tournament Analysis (Per Generation)**

```
INPUTS:
  - ALL strategies from generation (with metrics)
  - All debate transcripts
  - All logic ratings
  - Current market conditions
  - Historical lineage

OUTPUT:
  - Rankings (sorted by composite score)
  - Top N selections (strategies to evolve)
  - Variation prompts (mutations for next gen)
  - Forward projections
  - Archive recommendations
```

### Tournament Scoring

```python
composite_score = (
    0.30 × backtest_sharpe +
    0.20 × logic_rating_avg +
    0.30 × forward_projection +
    0.20 × confidence_score
)

# Normalize to 0-100 scale
# Sort descending
# Select top N
```

### Selection Criteria

**Multi-objective optimization:**
- Don't just pick highest Sharpe
- Select diverse strategies that are each "best" at something:
  - Best forward projection
  - Best logic + confidence
  - Best risk-adjusted backtest
  - Best market alignment
  - Most novel approach (exploration)

**Then evolve each separately** → different branches explore different objectives

---

## Allocation & Execution Strategy

### Recommended Approach: **85/15 Single Champion**

```
ALLOCATION:
  - 85% to tournament winner
  - 15% cash reserve

LIFECYCLE:
  - Hold for 7-10 days (fixed)
  - Early exit only if:
    - Confidence drops <50%
    - Regime invalidation
    - Emergency event

REBALANCING:
  - Only if allocation drift >25%
  - Or emergency triggered
  - Otherwise: HOLD
```

### Why Single Champion (Not 10 Strategies)?

**The tournament exists to find THE BEST strategy. So use it.**

- Tournament tests 315 strategies rigorously
- Debate refines logic
- Backtest validates performance
- Meta-LLM provides forward projection

**If you trust the process → Concentrate on the winner**

Spreading across 10 strategies:
- ❌ Dilutes the winner's edge
- ❌ Increases transaction costs
- ❌ Adds complexity
- ❌ Undermines the tournament's purpose

### Entry Protocol

```
Don't buy everything at once - reduce slippage

Hour 1: Limit order for 40% of position (0.2% below market)
Hour 3: Limit order for 40% more (0.15% below market)
Hour 5: Market order for final 20% (accept current price)

Result: Better average entry, lower market impact
```

### Exit Protocol

```
Normal exit (Day 7-10):
  - Exit over 2 days using limit orders
  - Set limits 0.15-0.2% above market
  - Minimize slippage

Emergency exit (confidence <50% or crisis):
  - Market orders immediately
  - Accept slippage as cost of risk management
```

### When to Rebalance

```
ONLY rebalance if:
  1. Position drift >25% from target (rare)
  2. Emergency exit triggered
  3. Allocation change >20% (e.g., confidence drops significantly)

Otherwise: HOLD

Why: Transaction costs matter
  - Every trade has slippage + spread + fees
  - Only trade when benefit clearly exceeds cost
```

---

## Risk Management & Adaptation

### Dynamic Allocation Based on Confidence × Regime

```python
def calculate_allocation(strategy_confidence, regime_stability):
    """
    Adaptive allocation based on two factors:
    - How good is the strategy? (confidence)
    - How stable is the environment? (regime)
    """

    # Base allocation from confidence
    if strategy_confidence >= 85:
        base = 0.90
    elif strategy_confidence >= 75:
        base = 0.75
    elif strategy_confidence >= 65:
        base = 0.60
    else:
        base = 0.40

    # Regime stability multiplier
    if regime_stability >= 0.8:      # Stable
        multiplier = 1.0
    elif regime_stability >= 0.6:    # Moderate
        multiplier = 0.85
    elif regime_stability >= 0.4:    # Elevated risk
        multiplier = 0.65
    else:                             # Crisis
        multiplier = 0.40

    final = base × multiplier
    return max(0.20, min(0.95, final))  # Floor/ceiling

# Examples:
# High conf (88%) + Stable (0.85) → 90%
# High conf (88%) + Unstable (0.35) → 36%
# Low conf (62%) + Stable (0.85) → 60%
```

### Regime Stability Score

```python
class RegimeStability:
    def calculate(self, market_data):
        scores = []

        # 1. Volatility level (VIX)
        if market_data.vix < 15:
            scores.append(1.0)    # Very stable
        elif market_data.vix < 20:
            scores.append(0.8)    # Stable
        elif market_data.vix < 30:
            scores.append(0.5)    # Elevated
        else:
            scores.append(0.2)    # Crisis

        # 2. Volatility trend
        vix_change = abs(market_data.vix - market_data.vix_5d_ago)
        if vix_change < 2:
            scores.append(1.0)    # Stable volatility
        elif vix_change < 5:
            scores.append(0.7)    # Moderate changes
        else:
            scores.append(0.3)    # Volatile volatility

        # 3. Trend clarity (ADX)
        if market_data.adx > 25:
            scores.append(1.0)    # Strong trend
        elif market_data.adx > 15:
            scores.append(0.7)
        else:
            scores.append(0.4)    # Choppy

        # 4. Correlation regime
        if market_data.spy_correlation < 0.5:
            scores.append(1.0)    # Normal
        elif market_data.spy_correlation < 0.8:
            scores.append(0.6)    # Elevated
        else:
            scores.append(0.2)    # Crisis correlation

        # 5. News/event risk
        news_risk = assess_news_risk(market_data.recent_news)
        scores.append(1.0 if news_risk == 'LOW' else
                     0.6 if news_risk == 'MODERATE' else 0.3)

        return sum(scores) / len(scores)
```

### Event Detection & Emergency Protocol

**Tier 1: Emergency (Immediate)**

```python
def check_emergency_events(market_data):
    emergencies = []

    if market_data.circuit_breaker_triggered:
        emergencies.append('CIRCUIT_BREAKER')

    if market_data.vix > 40 and market_data.vix_1h_change > 10:
        emergencies.append('VIX_SPIKE')

    if market_data.spy_intraday_change < -5:
        emergencies.append('FLASH_CRASH')

    if 'war' in market_data.breaking_news:
        emergencies.append('GEOPOLITICAL')

    return emergencies

# Action: Go 20% allocated, 80% cash immediately
```

**Tier 2: Rapid (Same Day)**

```python
def check_rapid_events(market_data, strategy):
    if strategy.regime_params_violated(market_data):
        return 'REGIME_INVALID'

    if strategy.current_confidence < 50:
        return 'CONFIDENCE_COLLAPSE'

    if abs(market_data.spy_change) > 3:
        return 'MAJOR_MOVE'

    return None

# Action: Run emergency tournament, transition over 1-2 hours
```

**Tier 3: Scheduled (Daily Morning)**

```python
# Normal daily update
regime_stability = calculate_regime_stability(market_data)
strategy_confidence = recalculate_confidence(strategy, market_data)
new_allocation = calculate_allocation(strategy_confidence, regime_stability)

if abs(new_allocation - current_allocation) > 0.10:
    rebalance_portfolio(new_allocation)
```

### Allocation State Machine

```
STATE 1: AGGRESSIVE (85-95%)
  Triggers: Confidence >85% AND Regime >0.8
  Allocation: 90% primary, 10% cash
  Goal: Maximize upside

STATE 2: CAUTIOUS (60-75%)
  Triggers: Confidence 70-85% OR Regime 0.6-0.8
  Allocation: 65% primary, 35% cash
  Goal: Balance growth and protection

STATE 3: DEFENSIVE (40-55%)
  Triggers: Confidence 60-70% OR Regime 0.4-0.6
  Allocation: 45% primary, 55% cash
  Goal: Preserve capital

STATE 4: CRISIS (20-35%)
  Triggers: Confidence <60% OR Regime <0.4 OR Emergency
  Allocation: 25% primary, 75% cash
  Goal: Survival
```

---

## Critical Flaws & Mitigations

### Flaw #1: Unproven System

**Problem:** You don't know if debates improve strategies, if Meta-LLM predicts accurately, or if tournament selects winners effectively.

**Mitigation:**
```
Phase 1 (30 days): Paper trading
  - Run system daily
  - Track predicted vs actual returns
  - Validate assumptions
  - NO real capital

Phase 2 (30 days): Small capital ($1k-$5k)
  - Real trades, real slippage, real emotions
  - Validate system works in practice

Phase 3: Scale only if proven
  - Gradual increase if profitable
  - Shut down if not working
```

### Flaw #2: Overfitting to Backtest

**Problem:** Evolution optimizes on historical data. Past performance ≠ future performance.

**Mitigation: Walk-Forward Testing**
```python
# Instead of backtesting full 2020-2024 period:

# Iteration 1:
train_period = '2020-01-01' to '2022-12-31'
test_period = '2023-01-01' to '2023-12-31'
# Only deploy strategies that work out-of-sample

# Iteration 2:
train_period = '2020-01-01' to '2023-12-31'
test_period = '2024-01-01' to '2024-12-31'
# Validate on fresh data

# Only trust strategies that work in multiple out-of-sample periods
```

### Flaw #3: Confidence Miscalibration

**Problem:** Meta-LLM says "85% confidence" but what does that actually mean?

**Mitigation: Build Calibration Curve**
```python
# After 100 strategies:
calibration_data = []

for strategy in completed_strategies:
    calibration_data.append({
        'stated_confidence': strategy.predicted_confidence,
        'actual_success': strategy.realized_return > 0
    })

# Analyze:
# Strategies rated 85% confidence → 72% actually succeeded
# Strategies rated 75% confidence → 65% actually succeeded

# Build calibration curve
# Adjust future allocations based on calibrated confidence
```

### Flaw #4: Evolutionary Convergence

**Problem:** Always evolving from winners leads to local optima and loss of diversity.

**Mitigation: Inject Diversity**
```python
# 80% of time: Evolve from tournament winners
# 20% of time: Random new seed (fresh exploration)

# Or maintain parallel tracks:
track_a = evolve_aggressive_growth_strategies()
track_b = evolve_defensive_strategies()
track_c = evolve_contrarian_strategies()

# Prevents monoculture
# Ensures robust strategy portfolio
```

### Flaw #5: Transaction Costs

**Problem:** Daily rebalancing could cost 15-20% annually in slippage/spreads/fees.

**Mitigation:**
```python
# 1. Minimum hold period: 10 days (not 7)
# 2. Only switch if benefit >3× transaction cost
# 3. Use limit orders (not market orders)
# 4. Track actual realized costs
# 5. Adjust if costs exceed 2% annually

# Example:
expected_benefit = (new_conf - current_conf) × 0.005
transaction_cost = 0.0035  # 35 bps
hurdle = transaction_cost × 3

if expected_benefit > hurdle:
    switch()
else:
    hold()
```

### Flaw #6: Regime Lag

**Problem:** Daily updates miss intraday events (Fed announcements, geopolitical crises, flash crashes).

**Mitigation: Continuous Monitoring**
```python
# Continuous checks for critical signals:
while market_open:
    if vix_spike > 10_points:
        emergency_reevaluation()

    if spy_move > 3_percent:
        emergency_reevaluation()

    if breaking_news in ['war', 'crash', 'emergency']:
        emergency_reevaluation()

    sleep(300)  # Check every 5 minutes
```

---

## Implementation Plan

### Phase 1: Build Core (Week 1-2)

```python
# 1. Evolution Framework
class EvolutionEngine:
    def run_generation(self, prompts, V, K):
        """Run one generation of evolution"""
        pass

# 2. Debate Framework
class DebateSystem:
    def run_debate(self, prompt, rounds=3):
        """LLM1 vs LLM2 debate"""
        pass

# 3. Meta-LLM
class MetaLLM:
    def synthesize_strategy(self, prompt, debate, market_data):
        """Create strategy from debate + context"""
        pass

    def tournament_analysis(self, strategies, market_data):
        """Rank and select winners"""
        pass

# 4. Composer Integration
class ComposerInterface:
    def create_symphony(self, strategy):
        pass

    def backtest_symphony(self, symphony_id, start_date, end_date):
        pass

# 5. Paper Trading
class PaperPortfolio:
    def track_position(self, strategy, allocation):
        pass

    def calculate_returns(self):
        pass
```

### Phase 2: Validation (Week 3-4)

```
Daily cycle for 20 days:
  1. Run full evolution (V=4, G=6, K=40%)
  2. Select tournament winner
  3. PAPER TRADE at 85% allocation
  4. Track predicted vs actual
  5. Log everything

Analysis:
  - Debate value-add: Pre vs post-debate Sharpe
  - Meta-LLM accuracy: Predicted vs actual returns
  - Tournament selection: Did winners actually outperform?
  - Confidence calibration: Build calibration curve
  - Cost measurement: Simulated transaction costs
```

### Phase 3: Small Capital (Week 5-6)

```
Deploy $2,000 real capital:
  - Simple rules: 85% winner, 15% cash
  - 10-day holding period
  - Limit orders for entry/exit
  - Track every trade
  - Measure actual slippage/spreads
  - Log emotional reactions
  - Document what goes wrong

Goal: Validate system works with real money and emotions
```

### Phase 4: Analysis & Refinement (Week 7-8)

```
Analyze results:
  - Was it profitable?
  - Were predictions accurate?
  - What failed? Why?
  - What transaction costs were incurred?
  - How did emotions affect decisions?
  - What would you change?

Refine:
  - Adjust confidence calibration
  - Tune allocation rules
  - Improve debate prompts
  - Fix broken components
```

### Phase 5: Scale or Scrap (Week 9+)

```
Decision point:

  IF profitable AND process worked AND predictions accurate:
    → Gradually scale capital
    → Continue iterating
    → Build meta-learning loop

  ELSE:
    → Fix identified issues
    → Return to validation phase
    → OR scrap and redesign
```

---

## Final Recommendations

### What I Would Actually Do

**Allocation:**
```
85% Tournament Winner
15% Cash

Simple. Trust the tournament.
```

**Holding Period:**
```
10 days (not 7)
  - Gives strategy time to work
  - Reduces transaction frequency
  - Lower costs

Early exit only if:
  - Confidence <50%
  - Regime invalidation
  - Emergency event
```

**Entry/Exit:**
```
Entry: Over 2 days with limit orders (reduce slippage)
Exit: Over 2 days with limit orders (maximize price)
Emergency: Market orders (speed over price)
```

**Rebalancing:**
```
Only if:
  - Allocation drift >25%
  - Or emergency triggered

Otherwise: HOLD
```

**Evolution:**
```
V=4, G=6, K=40%
  - 315 backtests
  - ~4 hours runtime
  - 64 final strategies
  - Run daily or every 2-3 days
```

### Key Principles

1. **Start small** - Don't bet the farm on v1
2. **Validate thoroughly** - Paper trade before real money
3. **Track everything** - Predicted vs actual, costs, decisions
4. **Be willing to fail** - Shut down if it doesn't work
5. **Trust the process** - If tournament is rigorous, use the winner
6. **Keep it simple** - Complexity = more failure modes
7. **Manage costs** - Transaction costs kill returns
8. **Build calibration** - Confidence scores need real-world validation

### What Would Surprise Me

**I predict:**
- Debate framework will add moderate value (10-20% improvement)
- Evolution will work well (clear improvement from Gen 0 to Gen 6)
- Meta-LLM forward projection will be the weak link (LLMs struggle with market prediction)
- Confidence scores will need heavy calibration (rarely accurate out of the box)
- Transaction costs will be higher than expected
- Emotional discipline will be harder than expected

**What would shock me:**
- Confidence scores accurate without calibration
- System profitable in first month
- No overfitting issues
- Meta-LLM predicts markets accurately

### Bottom Line

**This system is theoretically brilliant but practically unproven.**

The path forward:
1. Build it
2. Test it small
3. Measure everything
4. Scale only if proven
5. Be disciplined

**If it works:** You've built something genuinely novel - an evolutionary, self-improving investment system with LLM-powered strategy refinement.

**If it doesn't:** You'll learn exactly what went wrong and can iterate.

**But don't assume it works. Prove it.**

---

## Appendices

### A. Configuration Options Comparison

| Parameter | Conservative | Recommended | Aggressive |
|-----------|-------------|-------------|------------|
| Variations (V) | 2-3 | 4 | 5-6 |
| Generations (G) | 3-4 | 6 | 7-10 |
| Keep % (K) | 30-40% | 40% | 50-60% |
| Hold period | 14 days | 10 days | 7 days |
| Allocation | 70% | 85% | 95% |
| Evolution freq | Weekly | Daily | Daily |

### B. Cost Model

```
Assumptions:
  - Slippage: 20 bps (0.20%)
  - Spread: 10 bps
  - Commissions: 5 bps
  - Total per round trip: 35 bps

Annual costs:
  - 10-day hold → ~36 trades/year → 12.6% annual cost
  - With limit orders → ~8% annual cost
  - With 20% rebalance threshold → ~5% annual cost

Conclusion: Must optimize entry/exit and minimize turnover
```

### C. Walk-Forward Backtest Schedule

```
Rolling windows for validation:

Train: 2020-2021 (24 months)
Test:  2022 (12 months)

Train: 2020-2022 (36 months)
Test:  2023 (12 months)

Train: 2020-2023 (48 months)
Test:  2024 (12 months)

Only deploy strategies that work across multiple out-of-sample periods.
```

### D. Meta-Learning Feedback Loop

```python
# After each strategy completes lifecycle:
actual_return = strategy.realized_return
predicted_return = strategy.forward_projection
predicted_conf = strategy.confidence

feedback_entry = {
    'strategy_id': strategy.id,
    'predicted_return': predicted_return,
    'actual_return': actual_return,
    'prediction_error': abs(actual_return - predicted_return),
    'predicted_confidence': predicted_conf,
    'market_regime_at_entry': strategy.regime_snapshot,
    'success': actual_return > 0
}

# Feed back to Meta-LLM for learning
meta_llm.update_with_result(feedback_entry)

# Meta-LLM learns:
# "In low-VIX environments, my predictions are 15% too optimistic"
# "In high-correlation regimes, confidence should be reduced by 20%"
# Adjusts future predictions accordingly
```

### E. Emergency Protocol Checklist

```
TIER 1 EMERGENCY (Immediate Action):
  ☐ Circuit breaker triggered
  ☐ VIX spike >15 points in 1 hour
  ☐ SPY drop >5% intraday
  ☐ Geopolitical crisis (war, attack)
  ☐ Fed emergency meeting

  → Go to 20% allocation (80% cash) immediately

TIER 2 RAPID (Same Day Action):
  ☐ Regime parameters violated
  ☐ Confidence collapse (<50%)
  ☐ SPY move >3% intraday
  ☐ Major sector rotation

  → Run emergency tournament
  → Transition to new strategy over 2-4 hours

TIER 3 SCHEDULED (Next Day):
  ☐ Confidence decline (gradual)
  ☐ Regime stability decline
  ☐ Better alternative emerges

  → Update in morning routine
  → Gradual rebalancing
```

---

## Document Status

**Version:** 1.0
**Last Updated:** November 12, 2024
**Status:** Design Complete - Ready for Implementation
**Next Steps:** Begin Phase 1 (Build Core System)

**Key Decision Points:**
- ✅ Evolution parameters selected (V=4, G=6, K=40%)
- ✅ Allocation strategy defined (85/15 single champion)
- ✅ Debate framework architecture finalized
- ✅ Risk management protocols established
- ⏳ Implementation phase pending
- ⏳ Validation testing pending
- ⏳ Live deployment pending

**Open Questions:**
1. Which free LLM APIs to use? (Groq, Together AI, local?)
2. Composer API rate limits and costs?
3. Optimal debate prompt templates?
4. Meta-LLM prompt engineering for synthesis?
5. Real-world transaction costs vs estimates?

**Contact:** Ready for development kickoff
