# Polybot Third-Party Multi-Model Audit and Change Order SOP

**Document ID:** `POLYBOT-SOP-AUDIT-v1.1`
**Revision:** v1.1 (2026-11-16, Claude Fable 5.2, approved by Chase): added Stage 2b (author rebuttal); corrected identifiers, file names and paths checked against `snapshots/polybot_code_2026-11-14` + `CO-005.diff`; Stage 3 now tests against Hermes's current code and rejects destructive hunks. External citations and benchmark figures were not re-verified.
**Effective Date:** 2026-11-16
**Scope:** Standing governance, model-role assignment, blind-intake specification, adversarial reconciliation workflow, and Change Order (`CO`) lifecycle for Polybot (`polybot/` production runtime on Hermes `/opt/data/polybot` and `pipeline/` + `data/` + `models/` research repository on `<records>`).

---

## Executive Summary and Confidence Assessment

**Core Governance Rule (Medium-High Confidence):** No frontier model family that authored or tuned a Polybot strategy, ML pipeline, or Change Order may serve as the sole auditor or final approver of that artifact; independent sign-off requires (1) cross-family empirical data/pipeline verification on executable SQLite/Parquet data, (2) cross-family adversarial code/microstructure challenge, and (3) deterministic validation by Polybot's non-LLM `tools/scorecard.py` harness. Confidence that cross-vendor isolation plus executable verification catches materially more deployment defects than single-thread self-review is **medium-high** based on Polybot's own `CO-001` through `CO-006` defect record (`change_orders/`, 2026-10-25 to 2026-11-16) and primary empirical software-verification studies ([Jin & Chen, Automated Software Engineering, 2026-11-02](http://www.link.springer.com/article/10.1007/s10515-026-00638-5); [CrossAudit, arXiv preprint, 2026-09-03](https://arxiv.org/html/2608.28631)), while confidence in any fixed numerical ranking of frontier checkpoints is **low-to-medium** because public leaderboards do not directly measure prediction-market leakage or microstructure bug detection ([Artificial Analysis Intelligence Index v5.0.1 methodology, retrieved 2026-11-16](https://artificialanalysis.ai/evaluations/artificial-analysis-intelligence-index); [Song, arXiv preprint v2, 2026-10-06](https://arxiv.org/html/2610.01471)).

**Why This SOP Exists (What the Evidence Shows):**
1. **Authoring-session blind spots:** Claude Fable 5.2 authored `pipeline/`, `models/temperature_v1`, and `CO-005` (`change_orders/2026-11-14/CO-005-blend-strategy.md`, 2026-11-14), yet shipped a trade gate that opens at 15:00 station time (`m_lead == lead` through `tempmodel.PEAK_HOUR = 15`; `CO-005` had no `blend_min_local_hour` key) while citing a `late` backtest coded for `>= 18:00` (`pipeline/research_edge_after_costs.py` line 63, 2026-11-14), and ran that research replay with zero trading fees (the live `blend.py` did deduct `fee_rate * p * (1 - p)` from the edge) despite `69.6%` of historical lead-2 scans (`100%` since `2026-11-03`) carrying `fee_rate = 0.05` (`data/ml/market/blend_bucket_scan.parquet`, 2026-11-13).
2. **Prose-only audit blind spots:** An earlier third-party review (`change_orders/2026-11-16/Argon/AUDIT_CO-005_blend_2026-11-16.md`, 2026-11-16) ran while the local dataset was offline; it caught underpowered stopping rules (`F1`), missing `blend_snaps` columns (`F7`), and forecast staleness direction (`F4`), but missed the `15:00 vs 18:00` code mismatch, the unmodeled `0.05` fee regime, ascending-bucket crowding out, the dead `0.7843 < ask <= 0.90` price band, and `scorecard.py`'s after-the-fact `max(B1, B2)` baseline for `blend`, borrowed from the `forecast` universe.
3. **Correlated LLM errors without execution:** Multi-LLM research shows models share correlated failure modes when reasoning purely from text without executable data checks ([Kim et al., ICML/arXiv primary study, 2025-06-09 — older than 12 months; newer frontier checkpoints not tested](https://arxiv.org/pdf/2506.07962); [Correlated Errors in Multi-LLM Systems, Preprints.org v2, 2026-11-15](https://www.preprints.org/manuscript/202601.0892/v2); [Multi-Vendor Panel Blind Spots, arXiv preprint, 2026-07-19](https://arxiv.org/pdf/2605.12280)). Every Lead and Challenger audit in Polybot is therefore required to run deterministic DuckDB/SQLite/Python queries before issuing a Change Order.

---

## Model Role Assignment Matrix

Because Claude Fable 5.2 (`Anthropic`) is Polybot's primary architect and implementer (`CO-005-blend-strategy.md`, 2026-11-14), third-party audit duties are segregated across non-Anthropic frontier families (`Google DeepMind` and `OpenAI`), bounded support models, and non-LLM deterministic code.

| Role | Assigned Model / System | Lineage & Independence | Primary Audit & Change Order Responsibilities | Explicit Prohibitions |
|---|---|---|---|---|
| **Primary System Architect & Implementer** (and Fresh-Context Code Reviewer) | **Claude Fable 5.2** (`max` / high effort) ([Anthropic model announcement, 2026-11-09](https://www.anthropic.com/claude-fable-and-mythos-5-2); [Anthropic API docs, retrieved 2026-11-16](https://platform.claude.com/docs/en/models/fable-5-2/overview)) | Anthropic (Original Author — **Non-Independent** for self-authored work) | Designs `pipeline/` and `polybot/` features; drafts implementation diffs once Change Orders are approved; may run **clean-room, fresh-session** code review (`Fable-Clean`) to challenge third-party patches for syntax/integration regressions ([Artificial Analysis Fable vs Astra comparison, retrieved 2026-11-16](https://artificialanalysis.ai/models/comparisons/claude-fable-5-2-vs-gpt-6-astra)). | Never approves its own strategy, calibration claim, backtest, or `PREREGISTRATION.md` amendment; authoring thread context must never be reused for audit sign-off ([Self-Preference vs Capability, arXiv preprint, 2026-05-11](https://arxiv.org/html/2604.22891)). |
| **Lead Independent Quantitative, Data & Pipeline Auditor** | **Gemini 4 Carbon** (`high` / `max` thinking) ([Google DeepMind model card, 2026-11-12](https://deepmind.google/models/model-cards/gemini-4-carbon/); [Artificial Analysis Fable vs Carbon comparison, retrieved 2026-11-16](https://artificialanalysis.ai/models/comparisons/claude-fable-5-2-vs-gemini-4-carbon)) | Google DeepMind (**Independent Tier-1 Family**) | **Pillar 1 (Point-in-Time & Split Integrity)** and **Pillar 2 (Calibration, Proper Scoring Rules & Statistical Power)**: audits `pipeline/*.py`, `splits_v1.parquet`, `temp_hist_fc.parquet`, `blend_bucket_scan.parquet`, and `models/*/model.json`; executes full-dataset DuckDB/Python replays (`Terminal-Bench 4.0: 65.2%` per [Artificial Analysis, retrieved 2026-11-16](https://artificialanalysis.ai/models/comparisons/claude-fable-5-2-vs-gemini-4-carbon)); authors or supersedes quantitative and scoring Change Orders (`change_orders/YYYY-MM-DD/Carbon/`). | Must not issue a `CONFIRMED` quantitative verdict or strategy parameter change during a prose-only session when `data/ml/` or `polybot.sqlite` is offline; must label unexecuted checks `UNVERIFIED — DATA OFFLINE`. |
| **Independent Execution, Microstructure & Adversarial Code Challenger** | **GPT-6 Astra** (`max` effort) or **GPT-6.1 Sol** (`xhigh`/`max`) ([OpenAI GPT-6 Astra announcement, Nov 2026](https://openai.com/index/gpt-6-astra/); [OpenAI GPT-6.1 Sol announcement, Nov 2026](https://openai.com/index/introducing-gpt-6-1-sol); [MathArena leaderboard, 2026-11-14](https://matharena.ai/models)) | OpenAI (**Independent Tier-1 Family**) | **Pillar 3 (CLOB Microstructure, Fees, Sizing & Contract Resolution)** and **Pillar 4 (Adversarial Patch Verification & Seeded-Bug Testing)**: audits `broker.py`, `execution.py`, `clob.py`, `settle.py`, `stations.py`, `dashboard.py`, Kelly/cap math, Polymarket fee/tick/min-size rules, and SQLite state transitions (`FrontierCode-AA 2.0: 42.4%`, `CritPt v2 hard: 70.5%` per [Artificial Analysis, retrieved 2026-11-16](https://artificialanalysis.ai/models/comparisons/claude-fable-5-2-vs-gpt-6-astra)); challenges Carbon and Fable diffs (`change_orders/YYYY-MM-DD/Astra/`). | Must not rubber-stamp narrative reports or screenshot summaries as ledger truth; must verify exact SQLite schema, order/fill accounting invariants, and `--fuzz=0` patch application. |
| **Bounded Inventory, Log-Triage & Regression Runner** | **Gemini 4 Hydrogen** ([Google DeepMind model card, 2026-11-12](https://deepmind.google/models/model-cards/gemini-4-hydrogen/); [Oasis Carbon vs Hydrogen overview, Nov 2026](https://joinoasis.com/blog/gemini-4-hydrogen-vs-carbon)) | Google DeepMind (**Same Lineage as Carbon — Support Only**) | Rapid schema inventories, file/SHA-256 manifest generation, `py_compile` + unit-test sweeps, cron/refresh log anomaly extraction, and `test_vectors.json` parity checks. | Never counts as an independent vote alongside Gemini 4 Carbon (shared model family) and never signs off on a Change Order alone. |
| **Deterministic Non-LLM Judge (Final Gatekeeper)** | **`polybot/tools/scorecard.py` + `notes/SCORING_MANIFEST.json`** (Primary repo code, `snapshots/polybot_code_2026-11-14`) | Deterministic Python / SQLite (**Zero-LLM Arbiter**) | Computes date-clustered `t`-bands, effective dates (`N_eff`), fillable vs paper P&L, `B1`..`B4` do-nothing baselines (plus `B1_blend`/`B2_blend` and the `S1a`/`S1b`/`S2`/`S3` pre-registered `blend` gates, which exist only once a blend-evaluation Change Order is applied; the `2026-11-14` snapshot does not have them), and SHA-256 integrity of `scoring.py`, `governance.py`, `scorecard.py`, and `PREREGISTRATION.md`. | Cannot be edited without an explicit Change Order approved by Chase and re-registered via `tools/scorecard.py --register "<reason>"`. |

### Steelman and Caveats on Model Selection

1. **Benchmark provenance caveat (Medium Confidence):** In Artificial Analysis Intelligence Index v5.0.1 evaluations retrieved 2026-11-16 (secondary independent evaluator; single-harness source), Claude Fable 5.2 (`max`) leads overall (`Index 65`, `FrontierCode-AA 2.0: 48.8%`, `CritPt v2 hard: 76.5%`), Gemini 4 Carbon (`high`) leads agentic terminal execution (`Index 60`, `Terminal-Bench 4.0: 65.2%`), and GPT-6 Astra (`max`) leads Carbon slightly on coding and physics/math (`Index 57`, `FrontierCode-AA 2.0: 42.4%`, `CritPt v2 hard: 70.5%`) ([Artificial Analysis Fable vs Astra](https://artificialanalysis.ai/models/comparisons/claude-fable-5-2-vs-gpt-6-astra); [Artificial Analysis Fable vs Carbon](https://artificialanalysis.ai/models/comparisons/claude-fable-5-2-vs-gemini-4-carbon); [DataCamp secondary comparison, Nov 2026](https://www.datacamp.com/blog/claude-fable-5-2-vs-gemini-4-carbon)). Do not mix `FrontierCode-AA 2.0` with Cognition's native 132-task `FrontierCode 2.0` harness, which underwent harness rescoring on 2026-11-11 ([Cognition FrontierCode changelog, 2026-11-12](https://www.cognition.ai/frontiercode)).
2. **Optional additional challengers (Low Confidence for Polybot-specific lift):** `Grok 4.7` (`xAI`), `Kimi K3.1 Pro` (`Moonshot`), `DeepSeek V4.1` / `deepseek_v4_pro_0813` (`DeepSeek`), and `GLM-5.3` (`Z.ai`) appear on current public aggregators ([Artificial Analysis models list, retrieved 2026-11-16](https://artificialanalysis.ai/models); [LLMBenchmark rankings, 2026-11-15](https://llmbenchmark.ai/rankings); [BenchLeader Sol vs Astra comparison, Nov 2026](https://www.benchleader.com/compare/gpt-6-1-sol-vs-gpt-6-astra)). They may be used as tie-breakers if Carbon and Astra dead-lock, provided endpoint data-retention policies are verified first ([Anthropic Fable 5.2 retention docs, retrieved 2026-11-16](https://platform.claude.com/docs/en/models/fable-5-2/overview)).
3. **Orchestrator identity rule:** Inside Perplexity Computer, check which frontier model is active in `<session>` before counting an opinion as independent. A Perplexity session running GPT-6 Astra counts as the **Astra** review; a Perplexity session running Gemini 4 Carbon counts as the **Carbon** review. Never count the workspace wrapper and its underlying model as two separate auditors.

---

## Standard Folder Structure and Naming Convention

All audits and Change Orders must be stored in both the local research repository (`<records>/change_orders/`) and the Polybot Project repository (`change_orders/`) under date-and-model-scoped directories so competing or superseding reviews never overwrite each other:

```text
change_orders/
  README.md                                              # Master index of CO status (PROPOSED, SUPERSEDED, APPROVED, APPLIED, ROLLED_BACK)
  YYYY-MM-DD/
    <ModelCodename>/                                     # e.g., Carbon/, Astra/, Fable-Clean/, Argon/
      AUDIT_<CODENAME>_<TARGET>_YYYY-MM-DD.md            # Full 4-pillar skeptical audit report with reproducible scripts/SQL
      CO-NNN-<short-slug>.md                             # Change Order specification, risk/epoch impact, apply & rollback runbook
      CO-NNN.diff                                        # Unified diff applying with `patch -p1 --fuzz=0` from `/opt/data`
```

**Status Lifecycle Tags:**
- `DRAFT (UNVERIFIED — DATA OFFLINE)`: Written without executable access to `data/` or `snapshots/`. Cannot be deployed directly; must list concrete empirical verification tasks (`L1..Ln`).
- `PROPOSED (EMPIRICALLY VERIFIED)`: Verified against primary Parquet/SQLite data and tested with `patch -p1 --fuzz=0` + `py_compile` + unit replay. Awaits cross-model check or Chase's approval.
- `SUPERSEDED BY <Date>/<Model>/CO-NNN`: Retained intact for audit trail, never deleted, explicitly marked unimplemented.
- `APPROVED FOR HERMES`: Approved by Chase (`PREREGISTRATION.md` or strategy epoch change explicitly authorized).
- `APPLIED (SHA / MANIFEST REGISTERED)`: Live on `/opt/data/polybot` with `tools/scorecard.py --register` recorded in `notes/CHANGELOG.md`.

---

## Blind-Intake Evidence Bundle (Exact Polybot Path Mapping)

To prevent confirmation bias and anchor contamination ([Song, arXiv v2, 2026-10-06](https://arxiv.org/html/2610.01471)), every Stage 1 independent auditor receives **raw code, config, model weights, and data tables first** — withholding prior auditors' narrative conclusions until Stage 1 findings are locked. Map logical audit artifacts to exact Polybot paths:

| Audit Layer | Exact Polybot Paths (`<records>` & Hermes `/opt/data/polybot`) | Mandatory Integrity Checks |
|---|---|---|
| **1. Runtime Code & Config** | `snapshots/polybot_code_YYYY-MM-DD/polybot/polybot/*.py`, `config.json`, `cycle.sh`, `settle_fast.sh`, `tools/*.py`, `notes/PREREGISTRATION.md`, `notes/SCORING_MANIFEST.json`, `notes/CHANGELOG.md` | Verify `MANIFEST.json` SHA-256 hashes; check `config.py` defaults vs `config.json` overrides; verify `SCORING_MANIFEST.json` matches `tools/scorecard.py`. |
| **2. Live Ledger & Snapshots** | Hermes `/opt/data/polybot/data/polybot.sqlite` (exported via `data/raw/polybot_YYYY-MM-DD/` -> `data/curated/bot_order.parquet`, `bot_fill.parquet`, `bot_position.parquet`, `bot_cash.parquet`, `bot_config_epoch.parquet`, `mkt_snapshot.parquet`, `mkt_trade.parquet`, `mkt_meta.parquet`) | Verify `cash + settled_pnl - open_cost - open_fee` accounting identity; check `blend_snaps` / `snapshots` / `follow_snaps` schema via `PRAGMA table_info(...)`; inspect `orders WHERE status='rejected'`. |
| **3. Weather & Label Pipeline** | `pipeline/common.py`, `curated_weather.py`, `ml_temperature_hist.py`, `temperature_model.py`, `export_temperature_model.py`, `split_config.json`, `model_params.json`; `data/ml/splits_v1.parquet`, `data/ml/temperature/temp_hist_fc.parquet`, `temp_label_daily.parquet`, `data/curated/obs_metar.parquet` | Verify embargo gaps (`>= 7d`) in `splits_v1.parquet`; confirm `clim_c` uses strict `year < target_year`; check `trained_through` date in `model.json` vs `test` and `market_holdout` boundaries; verify `is_label_proxy` exclusions (`VHHH`, `RCSS`). |
| **4. Market Scan & Execution Tables** | `pipeline/curated_market.py`, `curated_books.py`, `research_speed_lag.py`, `research_edge_after_costs.py`; `data/ml/market/blend_bucket_scan.parquet`, `exec_resting_bid.parquet`, `data/curated/dim_market.parquet`, `dim_station.parquet` | Check `fee_rate` (`0.00` vs `0.05`), `min_size` (`5` shares), `yes_ask_depth` / `no_ask_depth`, `hours_to_day_start` vs station local hour, and `OUTAGE` windows (`2026-11-02 22:10Z` to `2026-11-04 03:31Z`). |
| **5. Deployed Model Package** | `models/temperature_v1/model.json`, `clim.json`, `predict.py`, `test_vectors.json`, `samples.json` (plus `change_orders/YYYY-MM-DD/*.diff` under review) | Run `predict.selftest()` (`360` vectors); verify whether `export_temperature_model.py` trained on `is_label_proxy` stations or refit across `test`; verify `var_scale` in-sample vs OOS residual calibration. |

---

## Four-Pillar Technical Audit Checklist

Every third-party audit must test all four pillars with executable code and classify each check as `CONFIRMED DEFECT`, `DISPROVED (CLEAN)`, `PLAUSIBLE / BOUNDED RISK`, or `UNTESTABLE WITH AVAILABLE DATA`.

### Pillar 1: Data Leakage, Lookahead Bias & As-Of Parity (Primary Owner: Gemini 4 Carbon)
1. **Forecast Publication vs Initialization Alignment:** Open-Meteo `previous_dayN` archives store what was predicted `24 * N` hours before valid time (`previous_runs` definition, [Open-Meteo Previous Runs API docs, retrieved 2026-11-16](https://open-meteo.com/en/docs/previous-runs-api)), whereas live `tempmodel.build(..., asof=False)` reads `K=0` (newest published run). Because Open-Meteo publishes global runs `5.9h` (`GFS`) to `7.7h` (`ECMWF`) after the run starts (the centres themselves release earlier) (`ANALYSIS_speed_lag_2026-11-13.md` section 6, 2026-11-13), auditors must verify whether live `K=0` at decision hour `H` is older or fresher than `previous_dayN`, and run a `previous_day(N+1)` staleness bracket on OOS data.
2. **Backtest Timing vs Live Config Gate:** Verify exact algebraic equality between backtest SQL filters (e.g., `hours_to_day_start <= 24 * (lead - 1) + 6` = `18:00` local) and the live gate actually in force. Read the code path, not only `config.py`: in `CO-005` the gate was `m_lead == lead` through `tempmodel.PEAK_HOUR = 15` with no config key, and `blend_min_local_hour` exists only once a Change Order adds it. Any discrepancy invalidates the backtest claim until the replay is rerun at the live gate.
3. **Split, Embargo & Refit Audit:** Check `splits_v1.parquet` (`train`, `val`, `test`, `market_holdout`, and `market_split`). Flag if a deployed model (`trained_through = "2026-10-09"`) absorbed `test` (`2026-03-08..2026-10-09`) before OOS reporting, or if strategy selection (`kind x lead x timing x w x threshold`) scanned the entire `market_holdout` without respecting `market_split` (`train`/`val`/`test`).
4. **In-Sample Variance / Calibration Scaling:** Inspect `temperature_model.py` (`fit()`) to check whether `sigma` or `var_scale` is fitted on out-of-fold residuals (`oof`) or in-sample tree residuals (`var` forest fitted on `X[ok]`, then scaled on the same `X[ok]`).

### Pillar 2: Calibration, Proper Scoring Rules & Statistical Edge (Primary Owner: Gemini 4 Carbon)
1. **Log-Loss vs Multiclass Brier Requirement:** Never accept winning-bucket log-loss (`-log p_win`) alone as proof of trading calibration. Log-loss heavily rewards rare tail adjustments (`0.01 -> 0.05`) while binary contract P&L at modal/favourite prices (`0.35..0.78`) tracks linear/quadratic probability accuracy. Every audit must compute **both** paired event log-loss (`S1a`) and **event multiclass Brier score** (`S1b = sum_k (p_k - y_k)^2`) on identical events with date-clustered `95%` intervals.
2. **Fair Baseline Decomposition:** When market probability definitions change (e.g., `PC-01` replacing `yes_ask` with `yes_ask / 2` on zero-bid buckets), decompose improvements into (a) quote-normalization gain alone, (b) standalone weather-model gain (`temperature_v1` vs `143`-member ensemble), and (c) equal-input log-pool gain (`new_blend` vs `old_ens + ask/2` at identical `w = 0.5`).
3. **Date-Clustered Power & Stopping Rules:** Weather errors correlate across stations on the same synoptic date (`N_eff` approx `15..25` dates, not `600+` trades). Every proposed `PREREGISTRATION.md` stopping rule must include a Monte Carlo power table at `0c`, `2c`, `4c`, and `6c` true per-share edge so futility and go-live gates have known Type I and Type II error rates.

### Pillar 3: Market Microstructure, Execution & Contract Resolution (Primary Owner: GPT-6 Astra)
1. **Fee-Exact Replay:** Check `dim_market.fee_rate` and `markets.fee_rate` (`taker_fee = shares * fee_rate * price * (1 - price)` in `polybot/broker.py`; `polybot/execution.py` is the resting-order path, maker fee `0`). Reject any research script (`research_edge_after_costs.py`) that assumes `fee = 0` on markets charging `fee_rate = 0.05`.
2. **Candidate Ordering & Cap Interactions:** Trace exact loop order in strategy evaluators (`blend.py`, `forecast.py`, `follow.py`). Check whether iterating buckets in ascending temperature order causes low-edge tail `NO` bets to consume `blend_max_event_usd` (`$4.00`) before higher-edge modal buckets are evaluated, and check interaction between `min_size` (`5` shares), `1.02` safety buffer, `blend_trade_usd` (`$2.00`), `blend_max_event_usd` (`$4.00`), and `blend_max_price` (`5 * 1.02 * p <= $4.00 => p <= 0.7843`).
3. **Order-Table Hygiene & Fill Realism:** Verify that persistent holds (`[held]`, `[opposite held]`, `[event cap]`) do not insert thousands of synthetic `status = 'rejected'` rows into `orders` on every 30-minute cron cycle, and check `yes_ask_depth` / `no_ask_depth` against requested share sizes and `scoring.fill_realism()`.
4. **Winner's Curse & Fractional Kelly Audit:** Compare average `p_pool` vs `price + fee` vs realized `win_rate` on executed signals. Measure edge shrinkage (`(realized_net_edge) / (claimed_edge)`) before permitting any move from flat `$2/$4` stakes to Kelly sizing.
5. **Station & Resolution Identity:** Verify `stations.py` and `is_label_proxy` (`VHHH` Hong Kong Observatory vs airport METAR; `RCSS` Taipei Civil Aeronautics vs Wunderground/CWA), station timezone DST handling (`ZoneInfo`), unit rounding (`0.5 F` vs `0.5 C` continuity endpoints), and dashboard mark-to-market freshness (for each `dashboard.py` price column, identify its source table and timestamp before calling it stale; `snapshots` and `follow_snaps` are still written after their strategies were retired).

### Pillar 4: Prioritized Change Orders & Governance Verification (Joint: Carbon + Astra + Scorecard)
1. **Priority Classification:**
   - **P0 (Immediate Blocker / Capital or Ledger Corruption):** Live order/accounting bugs, corrupted `scorecard.py` baselines or missing `blend` evaluation, unmodeled fee losses or wrong trading-window gates (`15:00 vs 18:00`), or unlogged decision gates (`blend_snaps`).
   - **P1 (Causal Backtest / Model Invalidation):** Candidate crowding-out bugs, unreachable price bands (`0.784..0.90`), stale dashboard MTM marks, silent API refresh failures, and `market_split` leakage.
   - **P2 (Next Model Retrain — `temperature_v2`):** Out-of-fold `var_scale` fitting, Student-`t` tail distribution replacement for Gaussian `ndtr`, publication-lag-matched training features, and API payload slimming (`blend_refresh_min` / forecast days).
   - **P3 (Cleanup / Refactoring):** Documentation, non-critical UI polish, archive pruning (`PC-15`).
2. **Patch & Epoch Discipline:** Every Change Order that modifies live entry selection, timing, price bounds, or candidate ordering **must** increment the strategy version string (e.g., `blend_version = "2026-11-16-v2"`) so `scoring.current_epoch_start()` (epochs come from a hash of the config keys starting `<strategy>_`, see `governance.strategy_keys` and `governance.epochs`) cleanly separates pre-fix and post-fix trades, and must include `tools/scorecard.py --register` instructions when `scoring.py`, `governance.py`, `scorecard.py`, or `PREREGISTRATION.md` changes.

---

## Multi-Model Workflow: Four-Stage Refute-or-Promote Protocol

### Stage 1: Blind Parallel Execution (No Cross-Contamination)
1. **Carbon Session:** Provide the Blind-Intake Bundle + the **Carbon Auditor Prompt Template** below. Carbon runs DuckDB/Python audits on `data/` and `snapshots/` and writes `AUDIT_CARBON_<TARGET>_<DATE>.md`.
2. **Astra Session:** Provide the identical Blind-Intake Bundle + the **Astra Challenger Prompt Template** below (without showing Carbon's output). Astra audits execution, CLOB math, SQLite invariants, and edge cases, writing `AUDIT_ASTRA_<TARGET>_<DATE>.md`.

### Stage 2: Empirical Reconciliation (Refute or Promote)
Compare Carbon and Astra findings line by line (plus any prior offline review such as `Argon`):
- **Agreement is not proof:** Even if both Carbon and Astra flag an issue from code reading, run the minimal DuckDB/SQLite query to quantify its exact dollar/nat impact.
- **Disagreement triggers a deciding query:** When Model A claims a defect and Model B disagrees (for example, Argon `F2` questioning whether `temperature_v1` beats the retired `143`-member ensemble on the same 23 dates), run a paired head-to-head script on identical `(scan_utc, event_id)` rows. Promote if empirically confirmed; mark `FALSIFIED BY PRIMARY DATA` with exact numbers if disproved.

### Stage 2b: Author Rebuttal (Evidence Only)
The authoring session (Claude Fable 5.2, with full project history) reviews every promoted finding and every proposed diff hunk before a superseding Change Order is built. Auditors work from a blind bundle and can misread code or data that the author knows in detail, so findings are contestable, under these rules:
- **Evidence or silence:** The author may contest a finding only with an executable query, a reproducible script, or an exact `file:line` citation. Argument from design intent alone is not a rebuttal.
- **Data wins:** A finding confirmed by primary data stands, whoever wrote the code. The author records it as an acknowledged mistake.
- **Contested findings go to the judge:** Where rebuttal and audit disagree, the deciding query is rerun on identical rows (Stage 2 rule) and the result is recorded with exact numbers as `CONFIRMED`, `OVERSTATED`, or `FALSIFIED BY PRIMARY DATA`.
- **Audit errors are logged too:** Hallucinated identifiers, wrong config keys, or destructive hunks found in an audit or its diff are listed in the rebuttal so they do not reach the Change Order.
- **No self-approval:** The author never approves their own work or their own rebuttal. Chase decides at Stage 4.

Output: `change_orders/YYYY-MM-DD/REBUTTAL_FABLE_<TARGET>_<DATE>.md`.

### Stage 3: Superseding Change Order Construction & Clean-Room Patch Test
The Lead Auditor (Carbon for model/scoring/strategy gates; Astra for broker/CLOB/settlement mechanics) builds a unified `CO-NNN.diff` and tests it in an isolated temp copy (`/tmp/polybot_verify/`):
1. Reconstruct exact pre-patch files from `snapshots/polybot_code_YYYY-MM-DD/` + prior applied diffs.
2. Verify `patch -p1 --fuzz=0 --dry-run < CO-NNN.diff` exits `0`.
3. Verify `python3 -m py_compile polybot/*.py tools/*.py` exits `0`.
3a. **Test against production, not only the snapshot:** request a fresh code export from Hermes (`tools/export_code.py`) and repeat steps 2 and 3 on it. Hermes edits files between snapshots (the dashboard changed after `2026-11-14`), so a diff that applies to the snapshot can fail or silently revert work on `/opt/data/polybot`. If no fresh export is available, the Change Order stays `PROPOSED` and says so.
3b. **Reject destructive hunks:** read every hunk header. A hunk that shrinks a locked document or a whole file (for example `@@ -1,96 +1,19 @@` on `notes/PREREGISTRATION.md`, which deletes sections 1 to 6) is a deletion, not an amendment, and blocks the Change Order unless Chase asked for that deletion. Amendments to `PREREGISTRATION.md` are append-only.
4. Run an end-to-end SQLite functional test exercising `evaluate_event()`, `PaperBroker`, `dashboard.build()`, `tools/scorecard.py --register`, and `tools/scorecard.py`.
5. Optional: pass `CO-NNN.diff` alone to a **fresh-context Claude Fable 5.2** session (`Fable-Clean`) with instructions to try to break the diff (checking imports, SQLite migrations, cron side effects, and rollback symmetry).

### Stage 4: Human Approval, Deployment & Post-Apply Audit
1. Chase reviews the Change Order summary, risk/epoch impact, and `PREREGISTRATION.md` diff.
2. Apply on Hermes (`/opt/data/polybot`), run one manual `./cycle.sh`, re-register `SCORING_MANIFEST.json`, and run the post-apply verification SQL queries specified in the `CO` document.

---

## Copy-Paste Auditor Prompt Templates

### Template A: Gemini 4 Carbon (Lead Quantitative, Data & Pipeline Auditor)

```text
Role: Independent Lead Quantitative, Data & Pipeline Auditor (Gemini 4 Carbon) for Polybot.
Context: Polybot's architecture, ML pipeline (`pipeline/`), `models/temperature_v1`, and Change Order under review were written by Claude Fable 5.2. Start from the skeptical hypothesis that the strategy, split hygiene, calibration claim, or Change Order is flawed, and test that hypothesis with executable Python/DuckDB/SQLite code against `<records>`.

Mandatory Tasks:
1. Pillar 1 (Leakage & As-Of Alignment): Audit `splits_v1.parquet`, `temp_hist_fc.parquet`, `temperature_model.py`, `export_temperature_model.py`, and `tempmodel.py`. Check `previous_dayN` vs live `K=0` publication lags, `trained_through` vs `test`/`market_holdout`, in-sample vs OOS `var_scale` fitting, and exact parity between backtest SQL timing filters and live `config.py` gates.
2. Pillar 2 (Calibration & Proper Scoring): Recompute BOTH event winning-bucket log-loss AND full-distribution multiclass Brier score (`sum_k (p_k - y_k)^2`) on identical `(scan_utc, event_id)` rows with date-clustered 95% intervals. Decompose quote-normalization effects (`ask/2`) from true model lift over the prior ensemble. Audit stopping-rule statistical power via simulation.
3. Pillar 3 (Trading Waterfall): Build a step-by-step replay waterfall from idealized research (`1 share, 0 fee, no caps`) to actual deployed `polybot` rules (real `fee_rate`, `$2/$4` caps, `min_size=5`, bucket iteration order, `max_price`). Compare against `market_split` (`train`/`val`/`test`) and live `polybot.sqlite` / dashboard behavior.
4. Pillar 4 (Change Order): Write `change_orders/YYYY-MM-DD/Carbon/AUDIT_CARBON_<TARGET>_<DATE>.md`, `CO-NNN-<slug>.md`, and `CO-NNN.diff` verified with `patch -p1 --fuzz=0`, `py_compile`, and an end-to-end SQLite test. Separate what the evidence shows, what you infer, and what you guess; attach High/Medium/Low confidence and inline dated citations to every claim; steelman the opposite conclusion; never use markdown italics or exclamation points.
```

### Template B: GPT-6 Astra / GPT-6.1 Sol (Execution, Microstructure & Code Challenger)

```text
Role: Independent Execution, Market Microstructure & Adversarial Code Challenger (GPT-6 Astra / Sol) for Polybot.
Context: Audit Polybot's execution, CLOB, broker, settlement, dashboard, and governance code (`polybot/*.py`, `tools/scorecard.py`) plus any proposed Change Order diff (`CO-NNN.diff`) without relying on author prose.

Mandatory Tasks:
1. Audit `execution.py`, `broker.py`, `clob.py`, `blend.py`, `forecast.py`, `follow.py`, and `settle.py` for taker/maker fee formulas (`fee_rate * p * (1 - p)`), `min_size` / `1.02` buffer / event-cap interactions, order-book depth (`yes_ask_depth`, `no_ask_depth`), queue/fill-realism bias, and `orders` table rejection spam.
2. Audit `stations.py`, `gamma.py`, and `settle.py` for contract resolution mismatches (proxy stations `VHHH`/`RCSS`, local timezone DST boundaries, `0.5` unit continuity bounds, early market closure).
3. Audit `scoring.py`, `governance.py`, `dashboard.py`, and `tools/scorecard.py` for baseline universe mismatches, epoch-boundary leaks (`scoring.current_epoch_start`, `governance.epochs`), stale mark-to-market pricing, and `SCORING_MANIFEST.json` integrity.
4. Attempt to break `CO-NNN.diff` with adversarial edge cases (NULL columns on legacy SQLite rows, network timeouts, zero-bid books, partial fills, `--reverse` rollback). Output `change_orders/YYYY-MM-DD/Astra/AUDIT_ASTRA_<TARGET>_<DATE>.md` with minimal failing/passing Python tests.
```

### Template C: Claude Fable 5.2 Clean-Room Integration Check (`Fable-Clean`)

```text
Role: Clean-Room Integration & Regression Reviewer (`Fable-Clean`, fresh session with zero authoring history).
Task: Inspect the third-party Change Order diff (`CO-NNN.diff`) against `snapshots/polybot_code_YYYY-MM-DD/`. You have no stake in prior design choices and no duty to accept the diff: neither defend the original code nor presume the audit is correct. Verify only: (1) `patch -p1 --fuzz=0` cleanliness, (2) SQLite schema migration idempotency on existing `/opt/data/polybot/data/polybot.sqlite`, (3) `cycle.sh` / `settle_fast.sh` / `dashboard.py` / `tools/scorecard.py` runtime compatibility and memory/API quota bounds, and (4) clean `patch -R -p1` rollback. Report only concrete code regressions with exact file:line citations.
```

---

## Cheapest Way to Verify This Governance Protocol Yourself

On the laptop, confirm both model-isolated directories (`Argon/` and `Carbon/`) coexist:

```bash
ls <records>/change_orders/2026-11-16/
```

Then check that a proposed diff applies with no fuzz on a throwaway copy of the snapshot plus the applied Change Orders:

```bash
T=$(mktemp -d) && cp -R <records>/snapshots/polybot_code_2026-11-14/polybot "$T/polybot" && cd "$T" \
  && patch -p1 -s < <records>/change_orders/2026-11-14/CO-005.diff \
  && patch -p1 --fuzz=0 --dry-run < <records>/change_orders/2026-11-16/Carbon/CO-006.diff
```

The `scorecard.py` manifest check needs the live ledger, so it runs on Hermes from `/opt/data/polybot`:

```bash
python3 tools/scorecard.py | head -n 25
```
