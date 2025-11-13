# Investor-AGI Evolutionary Engine – System Design

## 0. Purpose and Scope

This project implements an **evolutionary investing engine** powered by LLMs.

The system:

- Takes a **natural-language prompt describing an investing strategy**.
- **Backtests** the strategy.
- Runs a **debate**:
  - one LLM attacks it,
  - one LLM defends/improves the strategy,
  - across multiple rounds with **decreasing contentiousness**.
- Scores each argument with a **logic / reasoning score**.
- Feeds strategy, debate logs, scores, and backtest into a **Meta-LLM** that:
  - has access to **market data, past trends, news, and historical predictions/outcomes**,
  - synthesizes a refined strategy,
  - creates **strategy variations**,
  - produces forward **projections** and **hypotheses** about *why* the strategy works or fails.
- Repeats this process over **multiple generations**, keeping only the top fraction of strategies each time.
- At the end, runs a **tournament** over all strategies and picks a **champion** to trade.

Over time, the system **learns better reasoning** about markets because it:

- stores **all strategies, debates, scores, and outcomes**,
- forces the Meta-LLM to generate **explicit hypotheses**,
- tests those hypotheses against actual future results,
- updates its internal structure (hypotheses, weights, priors, and scoring rules).

This document defines the core data models, modules, and workflows needed to implement this.

---

## 1. High-Level Architecture

The system is organized into four layers:

1. **Data & Storage Layer**
   - Stores:
     - strategy prompts and specs,
     - episodes (full lifecycle of a strategy run),
     - market regime snapshots,
     - hypotheses and hypothesis updates.

2. **Evolution & Debate Engine**
   - Runs the evolutionary loop:
     - backtesting,
     - debates with decreasing contentiousness,
     - logic scoring,
     - Meta-LLM synthesis,
     - tournament selection,
     - multi-generation evolution.

3. **Cognition Layer (Learning & Reasoning)**
   - **Causal Hypothesis Layer**:
     - explicit market “rules of thumb” with belief weights.
   - **Meta-Reasoner**:
     - per-episode inference (why did this work/fail),
     - proposes hypothesis updates.
   - **Meta-Critic (Truth-Drift Detector)**:
     - cross-episode statistical analysis,
     - detects when hypotheses or calibration drift away from reality.
   - **Structural Risk Model**:
     - regime-aware allocation,
     - portfolio/risk guardrails.

4. **Interface Layer (CLI / Scripts)**
   - Commands to:
     - run one full evolutionary experiment (multi-generation),
     - run a single episode end-to-end,
     - run retrospective analysis / learning over past episodes.

---

## 2. Core Evolution Parameters

The default evolutionary configuration:

- **V: number of variations per strategy**  
  - Default: `V = 4` children per surviving strategy.
- **G: number of generations**  
  - Default: `G = 6` generations.
- **K: keep percentage per generation**  
  - Default: `K = 0.40` (keep top 40% of strategies each generation).

Process:

- Start with 1 or more **seed strategy prompts**.
- Each generation:
  - backtest + debate each strategy,
  - Meta-LLM synthesizes 1 refined version + `V` children per strategy,
  - tournament keeps top `K` fraction as seeds for the next generation.
- After `G` generations:
  - run a **global tournament** across all strategies,
  - pick a **champion** strategy (and possibly a ranked list).

---

## 3. Core Data Models

These will be implemented as Pydantic models or dataclasses in `src/data_models.py`.

### 3.1 StrategySpec

Represents a **strategy blueprint**.

- `id: str`
- `name: str`
- `prompt: str`  
  (original natural-language description)
- `description: str`  
  (refined explanation, may be updated by Meta-LLM)
- `parameters: dict`  
  (structured config for the backtester)
- `archetype: str`  
  (e.g. `"momentum"`, `"mean_reversion"`, `"breakout"`, `"stat_arb"`)
- `generation: int`  
  (0 for seeds, then 1..G)
- `parent_strategy_id: Optional[str]`  
  (for evolutionary lineage)
- `created_at: datetime`

### 3.2 DebateRound & DebateLog

**DebateRound**:

- `round_index: int`  
  (0-based)
- `contentiousness: float`  
  (1.0 = max conflict, 0.0 = minimal conflict; should decrease across rounds)
- `proponent_message: str`
- `critic_message: str`
- `logic_score: float`  
  (0–10, as given by a logic-rater LLM)

**DebateLog**:

- `strategy_id: str`
- `rounds: list[DebateRound]`

### 3.3 StrategyResult

Represents **backtested or realized performance** for a StrategySpec on a given window.

- `strategy_id: str`
- `episode_id: str`
- `metrics: dict`  
  (e.g. sharpe, max_drawdown, CAGR, win_rate)
- `equity_curve: Optional[list[float]]`
- `start_date: datetime`
- `end_date: datetime`
- `is_live: bool`  (false = backtest, true = live/paper)

### 3.4 RegimeSnapshot

Summarizes **market conditions** at decision/entry:

- `timestamp: datetime`
- `vix: float`
- `vix_change_5d: float`
- `adx: float`  
  (trend strength)
- `spy_corr: float`  
  (cross-asset correlation)
- `news_risk_level: str`  (`"LOW" | "MODERATE | "HIGH"`)
- `notes: Optional[str]`

### 3.5 MetaForecast

Meta-LLM’s **forward view** for a strategy at a given time.

- `strategy_id: str`
- `episode_id: str`
- `expected_return: float`
- `confidence: float`  (0–1)
- `horizon_days: int`
- `rationale: str`  
  (short explanation)
- `created_at: datetime`

### 3.6 Episode

One **full lifecycle** of a strategy run.

- `id: str`
- `strategy_spec: StrategySpec`
- `debate_log: DebateLog`
- `strategy_result: StrategyResult`  (backtest or realized)
- `regime_snapshot_at_entry: RegimeSnapshot`
- `meta_forecast: MetaForecast`
- `meta_inference: Optional[str]`  
  (post-hoc explanation of why the outcome occurred)
- `created_at: datetime`
- `is_live: bool`

### 3.7 Hypothesis

A **market rule or belief** that the system maintains explicitly.

- `id: str`
- `name: str`
- `description: str`  
  (e.g. “Short-term momentum in large-caps works best when VIX is 15–25 and ADX > 20.”)
- `applicable_conditions: dict`  
  (e.g. constraints on VIX/ADX, strategy archetypes, holding periods)
- `belief_weight: float`  (0.0–1.0)
- `created_at: datetime`
- `updated_at: datetime`

### 3.8 HypothesisUpdate

A record of how an Episode updates a Hypothesis.

- `id: str`
- `hypothesis_id: str`
- `episode_id: str`
- `delta_belief: float`  (positive or negative)
- `reason: str`          (short natural-language rationale)
- `created_at: datetime`

### 3.9 MetaCriticReport

Output of the **truth-drift detector**.

- `id: str`
- `generated_at: datetime`
- `hypotheses_to_down_weight: list[str]`
- `hypotheses_to_up_weight: list[str]`
- `calibration_issues: list[str]`  
  (e.g. “Overconfident in low-VIX regimes”)
- `regime_drifts_detected: list[str]`
- `notes: Optional[str]`

### 3.10 RiskAssessment & PortfolioRiskReport

**RiskAssessment** (per-strategy before execution):

- `strategy_id: str`
- `recommended_allocation: float`  (0.0–1.0)
- `risk_flags: list[str]`  
  (e.g. `"HIGH_VIX"`, `"HIGH_CORR"`, `"THIN_LIQUIDITY"`)
- `notes: Optional[str]`

**PortfolioRiskReport**:

- `generated_at: datetime`
- `total_risk_score: float`
- `flags: list[str]`
- `notes: Optional[str]`

---

## 4. Key Workflows

### 4.1 Generation & Debate Loop (per generation)

For a given generation `g`:

1. **Input:** a set of `StrategySpec` objects (seeds for this generation).
2. For each strategy:
   1. **Backtest** it on a designated in-sample window.
   2. Run a **Debate**:
      - Proponent LLM defends/improves the strategy.
      - Critic LLM attacks assumptions and edge cases.
      - Contentiousness starts high and **decreases each round**.
      - A separate “logic rater” LLM scores each round’s reasoning.
      - **Important:** debaters do *not* see backtest metrics.
   3. Store a `DebateLog` with `DebateRounds` and logic scores.
   4. Meta-LLM **synthesis**:
      - Inputs:
        - StrategySpec,
        - DebateLog + logic scores,
        - StrategyResult (backtest),
        - relevant market data, news, past predictions/outcomes,
        - applicable Hypotheses and recent MetaCriticReports.
      - Outputs:
        - a refined StrategySpec (same id),
        - a `MetaForecast`,
        - **V new StrategySpec children** (variations),
        - one or more **candidate Hypotheses** or updates.
3. **Tournament selection (within generation)**:
   - Compute a composite score for each strategy that may include:
     - backtest metrics,
     - average logic score,
     - MetaForecast expected return / confidence,
     - risk adjustments.
   - Keep top `K` fraction as seeds for next generation.
   - Collect all generated children from those winners as the next generation’s population.

Repeat this for `G` generations.

### 4.2 Final Tournament & Champion Selection

After `G` generations:

1. Gather all surviving `StrategySpec`s (and optionally the full lineage).
2. For each strategy:
   - Use latest:
     - backtest metrics on a validation/walk-forward window,
     - logic scores from debates,
     - MetaForecasts under current regime,
     - Structural Risk Assessment.
3. Meta-LLM runs a **global tournament analysis**:
   - ranks strategies,
   - considers diversification, robustness, regime fit,
   - selects:
     - **one champion strategy** for execution,
     - optionally a short ranked list of backups.
4. The champion strategy is what gets used for live/paper trading.

---

## 5. Retrospective Learning Workflow

For each completed **Episode** (backtest or live):

1. Retrieve all **Hypotheses** whose `applicable_conditions` match:
   - strategy archetype,
   - regime snapshot,
   - holding period.
2. Meta-LLM (Meta-Reasoner) generates:
   - `meta_inference` (why did the outcome occur?),
   - a set of `HypothesisUpdate` objects:
     - which hypotheses should be strengthened or weakened,
     - whether any new hypothesis should be added.
3. Store:
   - the Episode,
   - its `meta_inference`,
   - the associated `HypothesisUpdate`s.

Periodically (e.g. daily or weekly):

4. **MetaCritic** runs over a sliding window of Episodes & HypothesisUpdates:
   - checks performance of strategies where a hypothesis was “in play”,
   - looks at confidence vs realized success (calibration curves),
   - detects regime-dependent failures (e.g., momentum fails in crisis regimes).
   - emits a `MetaCriticReport` recommending:
     - hypotheses to down-weight or up-weight,
     - possible deletions/retirements,
     - calibration adjustments (e.g. “cut confidence by 20% in low-VIX environments”).
5. Apply changes to Hypotheses and store the `MetaCriticReport`.
6. For next generations and episodes:
   - pass relevant Hypotheses and MetaCriticReports to:
     - debates (as context / constraints),
     - Meta-LLM synthesis (as priors),
     - Structural Risk Model (as risk rules).

---

## 6. Structural Risk Workflow

Before live allocation:

1. Given:
   - a `StrategySpec`,
   - latest backtest and/or live `StrategyResult`,
   - current `RegimeSnapshot`,
   - current portfolio context,
   - relevant Hypotheses / MetaCriticReports,
   call `StructuralRiskModel` to compute a `RiskAssessment`.
2. The model:
   - may reduce recommended allocation based on regime risk,
   - may veto strategies that violate hard constraints (e.g., too much exposure in high-correlation crisis),
   - emits `risk_flags` and notes.
3. At the portfolio level:
   - occasionally run `PortfolioRiskReport` to check:
     - drawdown risks,
     - leverage and concentration,
     - exposure to specific failure regimes.

---

## 7. Module Mapping

Proposed Python module layout:

- `src/data_models.py`
  - StrategySpec, DebateRound, DebateLog, StrategyResult, RegimeSnapshot,
    MetaForecast, Episode, Hypothesis, HypothesisUpdate,
    MetaCriticReport, RiskAssessment, PortfolioRiskReport.

- `src/storage/store.py`
  - Simple JSON / SQLite persistence for all models.

- `src/engine/backtest_interface.py`
  - Abstractions for plugging in backtesting engines.

- `src/engine/debate_engine.py`
  - Runs Proponent vs Critic debates with decreasing contentiousness.
  - Ensures no backtest leakage to debaters.
  - Invokes a logic-rater LLM to produce logic scores.

- `src/engine/meta_llm.py`
  - Meta-Reasoner and Meta-LLM glue:
    - builds prompts using:
      - StrategySpec,
      - DebateLog,
      - StrategyResult,
      - Hypotheses,
      - MetaCriticReports,
      - market data / news.
    - outputs:
      - refined StrategySpec,
      - children StrategySpecs (variations),
      - MetaForecast,
      - HypothesisUpdates,
      - episode-level `meta_inference`.

- `src/engine/evolution.py`
  - Implements the V, G, K loop:
    - multi-generation evolution,
    - tournament selection per generation,
    - final champion tournament.

- `src/cognition/hypotheses.py`
  - `HypothesisStore`:
    - CRUD for Hypotheses,
    - retrieval of applicable hypotheses,
    - applying HypothesisUpdates.

- `src/cognition/meta_critic.py`
  - `MetaCritic`:
    - cross-episode analysis,
    - calibration checks,
    - generation of MetaCriticReports.

- `src/cognition/risk_model.py`
  - `StructuralRiskModel`:
    - per-strategy RiskAssessment,
    - portfolio-level PortfolioRiskReport.

- `src/cli/run_generation.py`
  - CLI entrypoint to run an entire evolutionary experiment.

- `src/cli/run_episode.py`
  - CLI entrypoint to run a single Episode end-to-end (from strategy prompt to Episode object).

---

## 8. Invariants and Design Principles

- **Debates do not see backtest metrics.**
  - Debaters must reason from strategy spec + priors, not hindsight.
- **Everything is logged as an Episode.**
  - Every strategy run (backtest or live) produces a complete Episode record.
- **Hypotheses are explicit, weighted, and updateable.**
  - Reasoning lives as objects, not just text transcripts.
- **Meta-Reasoner and Meta-Critic are separate.**
  - Meta-Reasoner: per-episode explanation + HypothesisUpdates.
  - Meta-Critic: cross-episode statistical drift detection.
- **Structural Risk Model is conservative.**
  - It can override “clever” strategies if they’re structurally dangerous.
- **Train/test / walk-forward separation must be respected.**
  - Evolution should run on in-sample windows,
  - tournament and champion selection should use out-of-sample / walk-forward data to avoid overfitting.
