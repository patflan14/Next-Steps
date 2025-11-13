# Investor-AGI Engine – System Design

## 0. Purpose and Scope

This project implements a **self-improving investment research engine** built around LLMs, debates, and retrospective learning.

The system:

- Generates and evolves **short- to medium-term trading strategies**.
- Uses **LLM debates** (without backtest leakage) to stress-test logic.
- Uses a **Meta-LLM** to synthesize a final strategy + confidence and projections.
- Executes or paper-trades strategies and logs results.
- Runs a **retrospective learning loop** that:
  - infers why results did what they did,
  - updates a **Causal Hypothesis Layer** (market “rules of thumb”),
  - uses a **Meta-Critic** to detect when those rules drift from reality,
  - and applies a **Structural Risk Model** to keep the whole thing from blowing up.

This is an **offline research / paper trading engine** at first. Later it can connect to real brokers.

Non-goals (for now):

- No actual order routing / brokerage integration.
- No full-blown distributed infrastructure.
- No fancy UI; simple CLI/scripts are fine.

---

## 1. High-Level Architecture

The system is organized into four conceptual layers:

1. **Data & Storage Layer**
   - Responsible for persisting:
     - strategies,
     - episodes (one full lifecycle of a strategy),
     - regime snapshots,
     - hypotheses,
     - hypothesis updates.
   - Backed by simple JSON or SQLite at first.

2. **Core Engine Layer**
   - Orchestrates the generation → debate → synthesis → tournament → execution pipeline.
   - Integrates with:
     - a backtest engine (external or stubbed),
     - an LLM client (for debates and meta-reasoning).

3. **Cognition Layer**
   - **Causal Hypothesis Layer**
     - Stores explicit, human-readable market hypotheses.
     - Updates their belief weights based on outcomes.
   - **Meta-Critic (Truth-Drift Detector)**
     - Monitors whether hypotheses and overall behavior still match reality.
   - **Structural Risk Model**
     - Enforces portfolio-level constraints and regime-aware risk limits.

4. **Interface Layer (CLI / Scripts)**
   - Commands to:
     - run one generation of strategy evolution,
     - run a full episode lifecycle,
     - run retrospective analysis over past episodes.

---

## 2. Core Data Models

Implementation detail: these will be Pydantic models or dataclasses in `src/data_models.py`.

### 2.1 StrategySpec

Represents a **strategy blueprint** before execution.

Key fields (subject to refinement):

- `id: str`
- `name: str`
- `description: str`  
  (natural language explanation of the idea)
- `parameters: dict`  
  (e.g., lookback windows, filters)
- `archetype: str`  
  (e.g., `"momentum"`, `"mean_reversion"`, `"breakout"`)
- `created_at: datetime`
- `parent_strategy_id: Optional[str]`  
  (for evolution lineage)

### 2.2 StrategyResult

Represents **backtest or realized performance** for a given StrategySpec on a given time window.

- `strategy_id: str`
- `episode_id: str`
- `metrics: dict`  
  (e.g., sharpe, max_drawdown, CAGR, win_rate)
- `equity_curve: Optional[list[float]]`  
  (optional for now)
- `start_date: datetime`
- `end_date: datetime`
- `is_live: bool`  
  (false = backtest, true = live/paper)

### 2.3 RegimeSnapshot

A summary of the **market regime** around entry/holding:

- `timestamp: datetime`
- `vix: float`
- `vix_change_5d: float`
- `adx: float`  
  (trend strength)
- `spy_corr: float`  
  (cross-asset correlation proxy)
- `news_risk_level: str`  
  (`"LOW" | "MODERATE" | "HIGH"`)
- `notes: Optional[str]`

This can be expanded later, but keep it simple at first.

### 2.4 Episode

A complete lifecycle of a strategy instance:

- `id: str`
- `strategy_spec: StrategySpec`
- `strategy_result: StrategyResult`  
  (backtest or live result)
- `regime_snapshot_at_entry: RegimeSnapshot`
- `meta_forecast: dict`  
  (Meta-LLM’s predicted return, confidence, etc.)
- `meta_inference: Optional[str]`  
  (post-hoc natural language reasoning about the outcome)
- `created_at: datetime`
- `is_live: bool`

### 2.5 Hypothesis

A **market rule of thumb** the system maintains explicitly.

- `id: str`
- `name: str`
- `description: str`  
  (e.g., “Short-term momentum works best in moderate volatility and strong trend conditions.”)
- `applicable_conditions: dict`  
  (e.g., ranges on VIX/ADX, strategy archetypes)
- `belief_weight: float`  
  (0.0–1.0, how strongly we currently believe this hypothesis)
- `created_at: datetime`
- `updated_at: datetime`

### 2.6 HypothesisUpdate

Records how an Episode influenced a Hypothesis.

- `id: str`
- `hypothesis_id: str`
- `episode_id: str`
- `delta_belief: float`  
  (positive or negative adjustment)
- `reason: str`  
  (short explanation)
- `created_at: datetime`

### 2.7 MetaCriticReport

Summary of the **truth-drift detector’s** findings over a window of episodes.

- `id: str`
- `generated_at: datetime`
- `hypotheses_to_down_weight: list[str]`
- `hypotheses_to_up_weight: list[str]`
- `regime_drifts_detected: list[str]`
- `notes: Optional[str]`

### 2.8 RiskAssessment / PortfolioRiskReport

Outputs of the Structural Risk Model.

**RiskAssessment** (per-strategy before execution):

- `strategy_id: str`
- `recommended_allocation: float`  
  (0.0–1.0 of capital)
- `risk_flags: list[str]`  
  (e.g., `"HIGH_VIX"`, `"HIGH_CORRELATION"`, `"EXCESSIVE_LEVERAGE"`)
- `notes: Optional[str]`

**PortfolioRiskReport** (portfolio-level):

- `generated_at: datetime`
- `total_risk_score: float`
- `flags: list[str]`
- `notes: Optional[str]`

---

## 3. Key Workflows

### 3.1 Generation & Debate Workflow (per generation)

1. **Seed or input initial StrategySpecs.**
2. For each spec:
   - Run **backtest** (stub at first).
   - Run **LLM debates** (Proponent vs Critic, no backtest leakage).
   - Collect debate transcripts + logic ratings.
3. Meta-LLM **synthesizes**:
   - one final strategy variant,
   - confidence score,
   - forward projection.
4. Tournament:
   - Rank strategies using backtest metrics, logic ratings, forward projection.
   - Select top K% to keep.
   - Generate V variations per winner for next generation.

This logic is largely defined in `docs/EVOLUTION_ENGINE.md`.

### 3.2 Episode Execution & Logging

For each chosen strategy:

1. Attach a **RegimeSnapshot** (market conditions at the decision).
2. Store Meta-LLM’s **forecast** (expected return, confidence).
3. Execute or paper-trade.
4. After holding period ends:
   - Fill in **StrategyResult** with realized metrics.
   - Construct an **Episode** with all of the above.
   - Persist to storage.

### 3.3 Retrospective Learning Workflow

For each completed Episode:

1. Retrieve applicable **Hypotheses** based on:
   - strategy archetype,
   - regime snapshot.
2. Meta-LLM generates a **meta_inference**:
   - Why did this work/fail?
   - Which hypotheses does this support or contradict?
3. From that inference, produce a set of **HypothesisUpdates**:
   - adjust belief weights,
   - possibly create new Hypotheses if a pattern appears.

Periodically (e.g. daily or weekly):

4. Run **MetaCritic**:
   - Scan Episodes + HypothesisUpdates.
   - Detect:
     - hypotheses whose performance is degrading,
     - calibration drift (confidence vs realized outcomes),
     - regime changes.
   - Emit a **MetaCriticReport** with recommended weight changes / alerts.

5. Update **Hypothesis** belief weights based on this report.

6. Feed updated Hypotheses & MetaCriticReports into:
   - the next generation’s **debates** (as context),
   - the **Meta-LLM** synthesis (as priors),
   - the **Structural Risk Model** if relevant.

### 3.4 Structural Risk Workflow

Before executing a strategy or allocating capital:

1. Gather **StrategySpec**, **StrategyResult** (backtest), and **RegimeSnapshot**.
2. Call **Structural Risk Model** to:
   - compute a **RiskAssessment**,
   - adjust recommended allocation,
   - possibly veto or flag certain strategies.
3. At the portfolio level:
   - periodically compute **PortfolioRiskReport** against:
     - global exposure limits,
     - max drawdown tolerance,
     - correlation / concentration constraints.

---

## 4. Module Mapping

Planned module structure (Python):

- `src/data_models.py`
  - All Pydantic models: StrategySpec, StrategyResult, Episode, RegimeSnapshot, Hypothesis, HypothesisUpdate, MetaCriticReport, RiskAssessment, PortfolioRiskReport.

- `src/storage/store.py`
  - Persistence layer (JSON / SQLite) for the models.

- `src/engine/backtest_interface.py`
  - Abstractions to plug in any backtester.

- `src/engine/debate_engine.py`
  - Functions/classes to run LLM debates (Proponent vs Critic, logic rater).

- `src/engine/meta_llm.py`
  - MetaReasoner:
    - constructs prompts for post-episode inference,
    - turns Episodes into HypothesisUpdates.

- `src/engine/evolution.py`
  - Implements evolutionary loop:
    - generations, variations, tournament selection.

- `src/cognition/hypotheses.py`
  - `HypothesisStore`:
    - CRUD for Hypotheses,
    - retrieving applicable hypotheses for an Episode,
    - applying HypothesisUpdates.

- `src/cognition/meta_critic.py`
  - `MetaCritic`:
    - analyzes trends in Episodes and HypothesisUpdates,
    - emits MetaCriticReports.

- `src/cognition/risk_model.py`
  - `StructuralRiskModel`:
    - risk assessments and portfolio risk reports.

- `src/cli/run_generation.py`
  - CLI entrypoint to run one generation of evolution.

- `src/cli/run_live_cycle.py`
  - CLI entrypoint to run a full Episode lifecycle (backtest or paper-trade).

---

## 5. Invariants and Design Principles

- **No backtest leakage into debates.**
  - Debaters see the strategy spec, not its backtest metrics.
- **All Episodes are logged.**
  - There is no “silent” decision; every strategy run ends in an Episode.
- **Hypotheses are explicit and updateable.**
  - No hidden magic: if the system is using a “belief,” it should live in Hypotheses.
- **MetaCritic is separate from MetaReasoner.**
  - MetaReasoner = per-episode explanation & updates.
  - MetaCritic = cross-episode statistical analysis & drift detection.
- **Structural Risk Model is conservative.**
  - It may veto or downsize trades even if they look good individually.
  - Survivability > maximum theoretical return.

---
