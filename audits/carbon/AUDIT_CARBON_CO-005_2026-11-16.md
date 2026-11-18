# Gemini 4 Carbon Independent Audit of `CO-005`, `temperature_v1`, and Unimplemented Argon `CO-006`

**Auditor:** Gemini 4 Carbon (Lead Independent Quantitative, Data & Pipeline Auditor)
**Audit Date:** 2026-11-16
**Artifacts Inspected (Primary Sources):**
- Local Mac research repository (`<records>`, files dated 2026-11-13 to 2026-11-16): `pipeline/*.py`, `models/temperature_v1/{model.json,clim.json,predict.py,test_vectors.json}`, `data/ml/splits_v1.parquet`, `data/ml/temperature/{temp_hist_fc.parquet,temp_label_daily.parquet}`, `data/ml/market/{blend_bucket_scan.parquet,exec_resting_bid.parquet}`, `data/curated/{dim_market.parquet,bot_fill.parquet,obs_metar.parquet}`, `data/reports/{edge_after_costs.md,arrival_alignment.md,DATA_AUDIT_2026-11-13.md}`, `snapshots/polybot_code_2026-11-14/`.
- User-supplied Change Orders & reviews: `CO-005-blend-strategy.md` and `CO-005.diff` (Claude Fable 5.2, 2026-11-14); `ANALYSIS_review_2026-11-13.md` and `ANALYSIS_speed_lag_2026-11-13.md` (2026-11-13); session transcripts `1..3` (2026-10-29 to 2026-11-13); unimplemented `change_orders/2026-11-16/Argon/{AUDIT_CO-005_blend_2026-11-16.md,CO-006-blend-evaluation-and-logging.md,CO-006.diff}` (2026-11-16); live dashboard screenshot audit (`audit_2026-11-16/screenshot_audit_results.json`, 2026-11-16).

---

## Executive Verdict and Confidence

**`CO-005` as deployed is flawed in its execution timing, fee modeling, candidate ordering, and official scoring harness, and the unimplemented Argon `CO-006` fixes only logging and text while leaving six live code/parameter defects untouched (High Confidence).** On identical primary market scans (`290,950` liquid lead-2 bucket-scans across `23` resolved dates, `data/ml/market/blend_bucket_scan.parquet`, 2026-11-13), `models/temperature_v1` genuinely improves winning-bucket log-loss over both the market (`-0.037` to `-0.043` nats, `95%` date-clustered band below zero) and the retired `143`-member ensemble at equal `w = 0.5` (`-0.030` to `-0.040` nats), **but achieves identically zero improvement in multiclass Brier score (`-0.001` to `+0.001`)** and loses money under `CO-005`'s actual deployed rules (`ROI = -2.9%`, `95%` band `-9.7%` to `+4.2%`, `PnL = -$50.09` across `601` trades) because `CO-005` traded 3 hours earlier than its own `>= 18:00` backtest, omitted Polymarket's `0.05` `fee_rate`, iterated buckets in ascending order (crowding out modal edges under the `$4` event cap), left `0.7843 < ask <= 0.90` unreachable, spammed `orders` rejections every 30 minutes, and left `tools/scorecard.py` unable to score `PREREGISTRATION.md` section 7.

---

## Separation of Evidence, Inference, and Guesswork

1. **What the primary evidence directly shows (High Confidence):**
   - `pipeline/research_edge_after_costs.py` (2026-11-14, line 62) coded `late` as `hours_to_day_start <= 24 * (s.lead - 1) + 6` (`<= 30h`, i.e. **`>= 18:00` station local time** at `lead = 2`) and hardcoded zero fees on line 18 (`no fees (weather markets charge none)`). `CO-005` (`polybot/config.py`, 2026-11-14) instead deployed `blend_min_local_hour = 15.0` (`>= 15:00` local), while `blend_bucket_scan.parquet` (`2026-11-13`) shows `fee_rate = 0.05` on `351,208 / 504,889` (`69.6%`) of lead-2 high-temperature scans and **`100%` of scans since `2026-11-03`**, and `data/curated/bot_fill.parquet` records `$14.16` in real taker fees paid across `386` `forecast` fills.
   - Across the `23` resolved dates using the exact deployed `models/temperature_v1/model.json` (`2026-11-14`), `new_blend` (`w = 0.5`) beats `market` (`PC-01` `ask/2`) on winning-bucket log-loss by `-0.043` nats (`95%` CI `-0.058` to `-0.028`, `19/23` dates) at `>= 15:00` knowable and `-0.037` nats (`-0.051` to `-0.022`, `17/23` dates) at `>= 18:00`, yet **multiclass Brier difference (`sum_k (p_k - y_k)^2`, `pool - market`) is `-0.001` (`-0.007` to `+0.006`) at `>= 15:00` and `+0.001` (`-0.005` to `+0.007`) at `>= 18:00`**.
   - Replaying `CO-005`'s exact loop (`>= 15:00` knowable inputs, real `fee_rate`, `$2` trade / `$4` event cap, `min_size = 5` shares, ascending bucket order) yields `n = 601` trades (`89.0%` `NO`, `$1,739.81` stake), `PnL = -$50.09` (`fee = $19.68`), **`ROI = -2.9%` (`-9.7%` to `+4.2%`)**, `128` dead-zone cap rejections (`0.7843 < ask <= 0.90`), and `1,194` event-cap skips that `CO-005` writes into SQLite `orders` as synthetic rejections on every scan.
   - Fixing the three live execution defects in superseding `CO-006` (`blend_min_local_hour = 18.0`, descending-edge candidate sorting, `blend_max_price = 0.78`) shifts the exact same 23-date replay from `-$50.09` (`-2.9%`) to **`+$32.70` (`+2.5%`, `95%` band `-7.3%` to `+13.6%`, `n = 454`)** under historical `fee_rate` and **`+$50.34` (`+4.3%`, `-6.5%` to `+16.2%`, `n = 400`)** under the current `0.05` fee regime, with `0` dead-zone rejections.
2. **What we infer from the evidence (Medium-High Confidence):**
   - `blend`'s log-loss edge comes almost entirely from **tail-bucket probability rescue** (lifting underpriced tail outcomes from `0.01..0.03` toward `0.04..0.08`, where `-log(p)` has steep curvature), whereas `89%` to `94%` of `blend`'s executable trades are **`NO` bets on modal/shoulder buckets (`price + fee` ~ `0.55`)**, governed by Brier/linear accuracy where `pool` and `market` tie. Moreover, conditional on `edge >= 0.08`, `~85%` of apparent edge disappears (`+10.3c` claimed edge -> `+1.6c` realized net edge in `3c`) due to adverse selection / Winner's Curse against faster weather bots (`data/reports/arrival_alignment.md`, 2026-11-13).
   - Even after `CO-006`'s `+$82.79` replay improvement, `23` dates (`95%` CI `-7.3%` to `+13.6%`) cannot establish positive true EV; `blend` must remain **strictly paper-only** under `CO-006`'s deterministic 30-date futility and 60-date pass gates.
3. **What remains an estimate / guess until live Hermes telemetry is read (Low Confidence):**
   - Exact Open-Meteo commercial billing units per 24 hours on Hermes (`/opt/data/polybot`): based on [Open-Meteo Pricing documentation (retrieved 2026-11-16)](https://open-meteo.com/en/pricing), we estimate `~20.2` weighted API calls per station refresh (`~700k/month` across `48` active stations at `55`-minute intervals), versus `CO-005`'s estimate of `3` calls (`~3,600/day`), because Open-Meteo does not document whether multi-model comma lists multiply variable counts identically across `/v1/forecast` and `previous-runs-api`.

---

## Reconciliation of Argon `CO-006` (`F1–F9` and `L1–L5`) Against Primary Data

Argon's audit (`AUDIT_CO-005_blend_2026-11-16.md`, 2026-11-16) was conducted while the Mac repository was offline. Executing Argon's deferred tasks `L1–L5` directly on `<records>` resolves every open question:

| Argon Item | Argon Offline Claim | Carbon Empirical Verdict on Primary Data (`2026-11-16`) | Action in Carbon `CO-006` |
|---|---|---|---|
| **F1 / L4** (20d stopping rule & scorer) | 20d fillable P&L band has `6%–16%` power at `4c` edge; proposed `S1–S3` text in `PREREGISTRATION.md` §7 but wrote **no code** in `scoring.py` or `scorecard.py`. | **CONFIRMED & EXTENDED (High Confidence):** Argon's `CO-006.diff` updated `PREREGISTRATION.md` without implementing `S1–S3` in `scoring.py`/`tools/scorecard.py`, and missed that log-loss `S1` passes (`-0.037` nats) even when multiclass Brier is `0.000`. | Implemented deterministic `scoring.blend_evaluation()` (`S1a` log-loss, `S1b` multiclass Brier, `S2` vs `forecast`, `S3` per-share signal P&L) directly inside `scoring.py` and `tools/scorecard.py`. |
| **F2 / L1** (Added value of `temperature_v1` vs old ensemble) | Claimed `new_blend` (`-0.04`) might not beat old `143`-member pool (`-0.034`), suspecting `0.006` nat gap was noise. | **FALSIFIED AT `>=15:00` AND `>=18:00`; CONFIRMED AT `strict` (High Confidence):** Argon compared across incompatible market baselines (`PC-01` `ask/2` vs old `ask`). `PC-01` alone cut market log-loss by `-0.055` to `-0.066` nats (`1.495 -> 1.440` at `>=18:00`). On identical events (`ev=841` at `>=18:00`), `temperature_v1` alone (`1.500`) beats old `143`-ens (`1.628`) by `-0.128` nats, and `new_blend` (`1.403`) beats `old_ens + ask/2` (`1.443`) by **`-0.040` (`-0.060` to `-0.021`)**. Under `strict` (`fc3+tmax_3`, `ev=871`), `S2_fair = +0.001` (zero gain). | Keep `temperature_v1`, enforce `blend_min_local_hour = 18.0` where `S2_fair = -0.040` nats holds, and score `S2` live in `scorecard.py`. |
| **F3 / L1** (Variant selection on `market_holdout`) | Suspected `23` unseen dates were mined for strategy settings. | **CONFIRMED (High Confidence):** `splits_v1.parquet` (`2026-11-13`) defined `market_split` (`train` 15d `2026-10-19..11-02`, `val` 4d `11-04..11-07`, `test` 4d `11-09..11-12`), but `research_edge_after_costs.py` ignored `market_split` and printed **36 trading cells + 6 log-loss cells** across all 23 dates combined. On `CO-005` (`2a`), `market_split` ROI is `+1.5%` (`train`), `-14.4%` (`val`), and `-1.4%` (`test`). | Treat all pre-`CO-006` backtests as in-sample for strategy rules; bump `blend_version` to `"2026-11-16-v2"` for a clean forward epoch. |
| **F4 / L2** (Forecast staleness live vs training) | Live `K=0` is older than `_previous_day2` (48h before valid time); feared `cov80` might drop `< 75%`. | **DIRECTION CONFIRMED; SEVERITY BOUNDED (High Confidence):** On OOS `test` (`2026-03-08..2026-10-09`, `n = 10,329`, fitted strictly on `train+val` to `2026-02-28`), feeding 24h-staler `_previous_day3` (with `regime = NaN` upper-bound stress) shifts `tmax_2` RMSE `1.290 -> 1.481 C`, bucket LL `1.599 -> 1.733` (`+0.134` nats), `cov80` `85.0% -> 80.5%` (stays `> 75%` because in-sample `var_scale = 3.79` over-inflated baseline `cov80` to `85.0%`), while tail `|z| > 2.5` nearly doubles (`1.21% -> 2.27%` vs Gaussian `1.24%`). | Fix `tempmodel.py` docstring; raise `blend_min_local_hour` to `18.0` (cutting staleness by 3h and aligning after `12Z` global model arrival); queue Student-`t` + OOS `var_scale` for `temperature_v2`. |
| **L3** (Live METAR completeness for `err_*` features) | Feared live `observations` table might lack `>= 20` METARs/day, turning `err_*` features NULL live. | **FALSIFIED (High Confidence):** Querying `data/curated/obs_metar.parquet` (`source = 'observations'`, 30d window) shows **48 of 48 non-proxy stations** have `>= 21` complete days (`>= 20` routine METARs/day; minimum `21` days at `KBKF`/`KDEN`). Also exposed a real bug in `blend.inputs()` where `now` ignored `now_utc`, breaking deterministic replay. | Fixed `blend.inputs(..., now=now_utc)` and deterministic `ts` in `blend.py`; added `_refresh_fail` reporting in `__main__.py`. |
| **F6** (Open-Meteo API call quota & `bom_access_global`) | Estimated `9–20` weighted calls/refresh vs `CO-005`'s `3`. | **CONFIRMED + NEW DEAD-MODEL FINDING (Medium-High Confidence):** `tempmodel.gather()` requests `41` days (`weight_time = 41/14 = 2.93`) across 3 endpoints (`~20.2` weighted calls/refresh per [Open-Meteo Pricing, retrieved 2026-11-16](https://open-meteo.com/en/pricing)). In addition, `bom_access_global` was retired by Open-Meteo on `2025-06-26` (source date older than 12 months; confirmed in `temp_hist_fc.parquet` where `fc_bom_c` is `100%` NULL after June 2025, capping live `n_models <= 8`). | Monitor 24h Hermes units; if `> 50%` quota, raise `blend_refresh_min` `55 -> 110`. Strip `bom_access_global` in `temperature_v2`. |
| **F7** (`blend_snaps` missing columns) | `blend_snaps` lacked `input_age_min`, `tradable`, `edge_yes`, `edge_no`, `fee_rate`, `model_name`, `blend_version`. | **CONFIRMED & EXTENDED (High Confidence):** Argon's 8 columns still omitted `local_hour`, `yes_ask_depth`, and `no_ask_depth` (`blend_bucket_scan.parquet` had `yes_ask_depth` but zero `no_ask_depth` logging, blinding `NO` liquidity analysis). | `CO-006` adds **11 columns** (`SNAP_EXTRA`) including `local_hour`, `yes_ask_depth`, and `no_ask_depth`. |
| **F8 / L5** (`scorecard.py` baseline & `fill_realism` bugs) | Suspected `baselines_forecast` used wrong market universe (`lows`, `lead 1`). | **CONFIRMED WORSE THAN SUSPECTED (High Confidence):** Inspecting `polybot/scoring.py` (`snapshots/polybot_code_2026-11-14`) proves three bugs: (1) `baselines_forecast()` reads retired `snapshots` across `lead IN (1,2)` and `highest + lowest` with zero fees (`1/price - 1`); (2) `scorecard.py` line 66 compares `blend` against `max(B1, B2)` **chosen ex-post**; (3) `scoring.fill_realism()` queries `FROM snapshots` instead of `blend_snaps`, so `blend`'s adverse-selection metric in `scorecard.py` fails to find `blend`'s own books. | Implemented `scoring.baselines_blend()` (`B1_blend`/`B2_blend` taker `ask + fee` on `blend_snaps` tradable universe), benchmarked `blend` against `B2_blend`, and routed `fill_realism(strategy='blend')` to `blend_snaps`. |

---

## Pillar 1: Data Leakage, Lookahead Bias & As-Of Alignment Audit

### 1.1 Split Hygiene, Proxy-Station Refit Drift, and In-Sample Variance Scaling
Inspecting `data/ml/splits_v1.parquet`, `pipeline/ml_temperature_hist.py`, `pipeline/temperature_model.py`, and `pipeline/export_temperature_model.py` (`2026-11-13..14`, primary repository code and Parquet tables) establishes:
- **Chronological weather splits are clean (`High Confidence`):** `temp_split` enforces 7-day embargoes (`train`: `2024-01-01..2025-06-30`, `547d`; embargo `7d`; `val`: `2025-07-08..2026-02-28`, `236d`; embargo `7d`; `test`: `2026-03-08..2026-10-09`, `216d`; embargo `7d`; `market_holdout`: `2026-10-17..2026-11-15`). `models/temperature_v1/clim.json` sets `_year = 2026` using strictly `year < 2026`, and ERA5 was excluded so live features match `historical-forecast-api`.
- **Deployed `model.json` absorbed `test` and both proxy stations (`High Confidence`):** `export_temperature_model.py` (`--through 2026-10-09`) refit `models/temperature_v1/model.json` across `train + val + test` (`n_train = 48,443` for `tmax_2`) and omitted `AND NOT is_label_proxy`, training on all `50` stations (including `VHHH` Hong Kong and `RCSS` Taipei), whereas `research_edge_after_costs.py` (`predictions()`) fit only on the `48` non-proxy stations. That seemingly minor training difference shifts 23-date `late` 1-share 0-fee ROI from `+3.9%` (`+$13.17`, `n = 648` in `edge_after_costs.md`) down to **`+1.4%` (`+$4.70`, `n = 656`)** for the actual deployed `model.json`.
- **In-sample `var_scale` calibration bug + heavy tails (`High Confidence`):** In `pipeline/temperature_model.py` (`fit()`, lines 75–78), `oof` residuals are properly computed via 5-fold date-grouped CV, **but `gbt_var` is then fitted on `X[ok]` and `var_scale` (`3.7901` for `tmax_2`) is computed in-sample on the exact same `X[ok]` predictions (`mean(resid^2) / mean(exp(pred_in_sample))`)**. Because regularized log-variance trees compress in-sample predictions, `var_scale` over-scales bulk `sigma` on OOS `test` (`cov50 = 56.9%` vs `50%`, `cov80 = 85.0%` vs `80%`), while standardized residuals `(y - mu) / sigma` remain strongly leptokurtic (**excess kurtosis `= +2.21`** on OOS `test`). Using Gaussian `ndtr` in `predict.bucket_probs()` therefore underprices extreme `> 2.5 sigma` weather busts whenever inputs go stale.

### 1.2 The `15:00` vs `18:00` Timing Mismatch and Publication-Lag Bracket (`L2`)
- **3-hour code vs Change Order bug (`High Confidence`):** `CO-005-blend-strategy.md` (§2.3) justified `blend_min_local_hour = 15.0` by citing `edge_after_costs.md`'s `late` replay. However, `pipeline/research_edge_after_costs.py` line 62 defines `late` as `s.hours_to_day_start <= 24 * (s.lead - 1) + 6`, which for `lead = 2` is `<= 30.0` hours before target day start — **18:00 station local time, not 15:00**. Between `15:00` and `18:00` local time at many stations (especially Europe/Americas relative to `12Z` ECMWF/GFS publication at `~18:00Z–20:00Z`, `data/reports/arrival_alignment.md`, 2026-11-13), `previous_day2` in `temp_hist_fc.parquet` already reflects afternoon forecast runs that had **not yet arrived** on Open-Meteo at `15:00` local.
- **OOS Staleness Bracket (`L2`, `High Confidence`):** Fitting `tmax_2` and `tmax_3` strictly on `train + val` (`<= 2026-02-28`) and scoring on sealed OOS `test` (`2026-03-08..2026-10-09`, `n = 10,329` non-proxy highs):

| OOS Model & Input Vintage (`2026-03-08..2026-10-09`) | Rows | RMSE (`C`) | Bucket Log-Loss | `cov50` | `cov80` | `cov95` | Tail `|z| > 2.5` (Gaussian `1.24%`) | Excess Kurtosis |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `tmax_2` fed `_previous_day2` (48h before valid) | 10,330 | `1.290` | `1.599` | `56.9%` | `85.0%` | `96.0%` | `1.21%` | `+2.21` |
| `tmax_2` fed `_previous_day3` (72h + `regime=NaN` upper stress) | 10,329 | `1.481` | `1.733` | `52.4%` | `80.5%` | `94.3%` | **`2.27%`** | `+1.48` |
| `tmax_3` fed `_previous_day3` (72h matched model) | 10,329 | `1.474` | `1.726` | `54.3%` | `83.1%` | `95.5%` | `1.52%` | `+1.63` |

**Caveat:** In `temp_hist_fc.parquet`, `lead_days = 3` leaves `gfs_*`/`ecmwf_*` regime columns `NaN`, whereas live `K=0` populates them from the 72h run; row 2 is therefore a conservative upper-bound stress test combining 24h staleness plus missing regime features. Even with bulk `cov80` buffered at `80.5%`, 24 hours of staleness degrades bucket log-loss by **`+0.134` nats** (`3.6x` the entire `-0.037` nat edge over the market) and pushes `> 2.5 sigma` tail misses to `2.27%` (`1.83x` normal). Delaying entry from `15:00` to `18:00` local (`CO-006`) directly removes 3 hours of staleness and aligns execution with `research_edge_after_costs.py`'s actual `late` window.

---

## Pillar 2: Calibration, Proper Scoring Rules & The Log-Loss vs Brier Divergence

Scoring the **exact deployed `models/temperature_v1/model.json`** across all liquid lead-2 resolved non-proxy high-temperature events (`23` dates, `2026-10-20..2026-11-12`, paired date-clustered 95% bootstrap bands; `[sig]` denotes 95% band excluding zero):

| Timing Window (`lead = 2`, `tmax`, `23` dates) | Events | Market (`ask/2`) LL | Market (old `ask`) LL | `temp_v1` Alone LL | Old `143`-Ens LL | `new_blend` (`w=0.5`) LL | `S1a`: `blend - mkt(ask/2)` Log-Loss | `S2_fair`: `new_blend - old_ens(ask/2, w=0.5)` | `S1b`: `blend - mkt(ask/2)` **Multiclass Brier** |
|---|---:|---:|---:|---:|---:|---:|---|---|---|
| `strict` (all hrs, `fc3 + tmax_3`) | 871 | `1.511` | `1.600` | `1.598` | `1.629` | `1.468` | `-0.043 (-0.061 to -0.026)` [sig] (`18/23d`) | **`+0.001 (-0.021 to +0.023)`** | **`-0.001 (-0.007 to +0.006)`** |
| `CO-005` (`>=15:00` knowable: `fc3` 15–18h, `fc2` `>=18h`) | 856 | `1.464` | `1.530` | `1.528` | `1.627` | `1.420` | `-0.043 (-0.058 to -0.028)` [sig] (`19/23d`) | `-0.030 (-0.049 to -0.011)` [sig] | **`-0.001 (-0.007 to +0.006)`** |
| `CO-005` (`>=15:00` optimistic `fc2 + tmax_2`) | 856 | `1.464` | `1.530` | `1.496` | `1.627` | `1.408` | `-0.055 (-0.071 to -0.039)` [sig] (`19/23d`) | `-0.042 (-0.061 to -0.024)` [sig] | `-0.003 (-0.009 to +0.003)` |
| **`CO-006` `late` (`>=18:00`, `fc2 + tmax_2`)** | **841** | **`1.440`** | **`1.495`** | **`1.500`** | **`1.628`** | **`1.403`** | **`-0.037 (-0.051 to -0.022)` [sig] (`17/23d`)** | **`-0.040 (-0.060 to -0.021)` [sig]** | **`+0.001 (-0.005 to +0.007)`** |

**Why Log-Loss Improves While Multiclass Brier Is Zero (High Confidence):**
1. **Quote normalization (`PC-01`) did more work than the ML model:** Replacing full `yes_ask` with `yes_ask / 2` on zero-bid buckets reduced raw market log-loss by `-0.055` to `-0.089` nats on its own (`1.495 -> 1.440` at `>= 18:00`).
2. **Log-loss vs Brier sensitivity:** Winning-bucket log-loss (`-log p_win`) has slope `-1/p`; nudging an under-quoted tail outcome from `p = 0.02` to `p = 0.06` saves **`1.10` nats** on a tail upset while costing only `~0.04` nats when a `0.50` favourite wins. Multiclass Brier (`sum_k (p_k - y_k)^2`), by contrast, weights probability errors quadratically across all buckets and is dominated by the modal `0.30..0.70` buckets. Because `blend` achieves **zero multiclass Brier improvement (`+0.001` at `>= 18:00`)**, `blend` is **not** beating the market on modal/favourite probabilities — yet `89.0%` (`CO-005`) to `93.6%` (`CO-006`) of `blend`'s trades are `NO` bets on those exact modal/shoulder buckets (`avg price + fee = 0.542..0.554`). That explains why a genuine `-0.04` nat log-loss edge translates into near-zero or negative binary trading ROI once spreads and `5%` fees are charged.

---

## Pillar 3: Market Microstructure, Fee Waterfall, Execution Bugs & Dashboard MTM

### 3.1 Step-by-Step Reality Waterfall (`models/temperature_v1/model.json`, `23` Dates)
Replaying the exact `models/temperature_v1` predictions scan-by-scan through `blend_bucket_scan.parquet` (`290,950` rows) isolates every execution distortion separating `edge_after_costs.md` from live `CO-005` and superseding `CO-006`:

| Variant (`lead = 2`, `tmax`, `23` resolved dates) | Trades (`n`) | `NO` Share | Mean `p_pool` | Mean `px + fee` | Realized `won` | Date-Clustered ROI (`95%` CI) | Net P&L (Fees Paid) | Dead `>0.784` / Cap Skips |
|---|---:|---:|---:|---:|---:|---|---:|---:|
| `0a.` Idealized `late` (`>=18:00`, `1sh`, `0` fee, no caps) | 656 | `86.1%` | `0.629` | `0.526` | `0.534` | `+1.4% (-5.8% to +8.8%)` | `+$4.70 ($0.00)` | `0 / 0` |
| `0b.` Idealized `strict` (all hrs, `fc3`, `1sh`, `0` fee) | 1,083 | `87.6%` | `0.656` | `0.548` | `0.534` | `-2.5% (-7.3% to +2.6%)` | `-$15.00 ($0.00)` | `0 / 0` |
| `1a.` `>=18:00` + actual `fee_rate`, zero-fee signal (`1sh`) | 656 | `86.1%` | `0.629` | `0.533` | `0.534` | `+0.1% (-6.9% to +7.8%)` | `+$0.50 ($4.20)` | `0 / 0` |
| `1b.` `>=18:00` + `0.05` fee regime, zero-fee signal (`1sh`) | 656 | `86.1%` | `0.629` | `0.537` | `0.534` | `-0.7% (-7.7% to +6.6%)` | `-$2.46 ($7.16)` | `0 / 0` |
| `1c.` `>=18:00` + `0.05` fee regime + fee-aware edge (`1sh`) | 485 | `85.8%` | `0.631` | `0.530` | `0.551` | `+4.0% (-5.7% to +14.5%)` | `+$10.16 ($5.29)` | `0 / 0` |
| **`2a.` DEPLOYED `CO-005` (`>=15:00` knowable, real fee, `$2/$4`, asc)** | **601** | **`89.0%`** | **`0.649`** | **`0.542`** | **`0.531`** | **`-2.9% (-9.7% to +4.2%)`** | **`-$50.09 ($19.68)`** | **`128 / 1,194`** |
| `2b.` DEPLOYED `CO-005` under `0.05` fee regime (`>=15:00`, asc) | 541 | `89.8%` | `0.657` | `0.550` | `0.538` | `-2.5% (-9.7% to +5.4%)` | `-$39.88 ($33.73)` | `99 / 1,019` |
| `3a.` `>=15:00` knowable + `sort_by_edge` + `max_px=0.78` | 596 | `93.0%` | `0.668` | `0.560` | `0.544` | `-2.4% (-9.3% to +5.1%)` | `-$41.56 ($19.49)` | `0 / 1,182` |
| `3b.` `>=18:00` + ascending order + `max_px=0.90` (real fee) | 456 | `90.4%` | `0.637` | `0.536` | `0.548` | `+1.3% (-8.4% to +11.7%)` | `+$16.93 ($15.04)` | `66 / 593` |
| **`3c.` CARBON `CO-006` (`>=18:00` + `sort_by_edge` + `max_px=0.78`, real fee)** | **454** | **`93.6%`** | **`0.653`** | **`0.550`** | **`0.566`** | **`+2.5% (-7.3% to +13.6%)`** | **`+$32.70 ($14.98)`** | **`0 / 596`** |
| **`3d.` CARBON `CO-006` under `0.05` fee regime (`>=18:00` + `sort_by_edge`)** | **400** | **`93.5%`** | **`0.658`** | **`0.554`** | **`0.580`** | **`+4.3% (-6.5% to +16.2%)`** | **`+$50.34 ($24.73)`** | **`0 / 491`** |

### 3.2 Six Concrete Microstructure, Execution & UI Defects Identified
1. **Unmodeled `0.05` Taker Fee Regime (`High Confidence`):** Polymarket charges `fee = shares * fee_rate * p * (1 - p)` (`polybot/execution.py` line 6). At `p = 0.55` and `fee_rate = 0.05`, the taker fee is `1.24c/share` (`~2.25%` of stake), erasing `89%` of `0a`'s `+$4.70` gross profit when signals ignore fees (`1a`), though `blend.py`'s live `taker_fee` filter (`1c`, `3c`, `3d`) avoids the worst marginal trades.
2. **Asymmetric `YES` Longshot Toxicity (`High Confidence`):** Breaking down deployed `CO-005` (`2a`, `n = 601`, `$1,739.81` stake):
   - **`NO` trades (`n = 535`, `$1,605.72` stake, `92.3%` of capital):** `p_pool = 0.696`, `px + fee = 0.588`, realized `won = 0.579`, `PnL = -$23.72` (`ROI = -1.5%`).
   - **`YES` trades (`n = 66`, `$134.09` stake, `100%` non-favourite buckets):** `p_pool = 0.273`, `p_mkt = 0.150`, `px + fee = 0.176`, realized `won = 0.136`, **`PnL = -$26.37` (`ROI = -19.7%`)**. Where Gaussian `temperature_v1` disagrees sharply with a low market ask on a non-favourite `YES` bucket, the market is almost always pricing synoptic/local information the daily GBT lacks.
3. **Ascending-Bucket Crowding Out + Dead `0.7843..0.90` Price Band (`High Confidence`):**
   - `blend.py` iterates `ev["markets"]` in ascending temperature order (`lo` low to high). Because `min_size = 5` shares with `1.02x` buffer (`blend.py` line 156) scales a `$2.00` order on a `NO @ 0.75` shoulder bucket up to `5 * 0.75 * 1.02 = $3.83`, the **first** low-temperature bucket with `edge = 0.081` consumes `$3.83` of the `$4.00` `blend_max_per_event` cap, blocking a `0.19` edge on the central modal bucket (`+$15.77` drag between `3b` and `3c`, and verified deterministically in our SQLite unit test `Test B`).
   - Whenever `ask > $4.00 / (5 * 1.02) = 0.78431`, `min_usd > $4.00`, so `usd` stays `$2.00` (`< 5` shares) and `broker.execute` rejects the order every 30 minutes (`128` market-sides in `2a`). Lowering `blend_max_price` from `0.90` to `0.78` eliminates this unreachable dead zone cleanly.
   - Similarly, `blend.py` passed already-held (`[held]`), `[opposite held]`, and `[event cap]` candidates into `broker.reject()` on every 30-minute scan (`1,194+` rejections in replay), polluting the `orders` table. `CO-006` skips these before `broker.execute()`.
4. **Winner's Curse (`~85%` Edge Shrinkage) Forbids Kelly Sizing (`High Confidence`):** Even in `CO-006` (`3c`), `blend` claims `p_pool - (px + fee) = 0.653 - 0.550 = +10.3c/share` edge, yet realizes `0.566 - 0.550 = +1.6c/share` (`84.5%` shrinkage; `76%` shrinkage in `3d`). Full Kelly at claimed `p_pool = 0.653` vs `0.550` would stake `~23%` of bankroll (`$230/trade`), whereas empirical Kelly at `0.566` is `~3.6%` (`6.4x` smaller). `blend` must remain on tiny flat `$2/$4` paper stakes.
5. **CLOB Depth Ceiling (`High Confidence`):** Across all `869` `YES` signal scans (`edge >= 0.08`), median `yes_ask_depth` is **`39.1` shares** (`20.1%` have `< 5–11` requested shares; `63.5%` have `< 50` shares / `~$30`), and `blend_bucket_scan.parquet` failed to record `no_ask_depth` at all (`CO-006` adds both `yes_ask_depth` and `no_ask_depth` to `blend_snaps`).
6. **Live Nov 16 Validation & `dashboard.py` Stale Same-Day MTM Bug (`High Confidence`):** Chase's live dashboard on `2026-11-16` (`audit_2026-11-16/screenshot_audit_results.json`, 2026-11-16) shows `40` open `blend` positions (`37 NO / 3 YES = 92.5% NO`, `$108.92` cost, `-$9.27` / `-8.51%` displayed MTM; `$2.50` cost-vs-MTM gap matches `0.05 * cost * (1 - paid) = $2.49` taker fees). Furthermore, 8 same-day (`2026-11-15/16`) positions displayed `23.5pp` to `59.8pp` discrepancies between `Mark` (`0.001..0.004` or `0.995..1.000`) and `Market now` (`0.43..0.73`). Inspecting `polybot/dashboard.py` (`snapshots/polybot_code_2026-11-14`, lines 48–75 and 172–179) reveals the exact code bug: `_latest_books()` queried only `snapshots` and `follow_snaps` (omitting `blend_snaps`) and preferred stale lead-1 `yes_bid`/`no_bid` even on settlement day (`lead <= 0`) when the bot stopped scanning lead-0 books while `capture.py` continued updating `market_meta.mid_yes` (`gamma_yes`) hours later. `CO-006` fixes both in `dashboard.py`.

---

## Steelman of the Opposite Conclusion

**Strongest case that `CO-005` was sound and Carbon is overly skeptical:**
1. Retiring `forecast` (`-$2.13` over `295` settled trades; lost `21/21` dates at lead 1, `CO-005` §2.1) and moving to paper-only `blend` on lead-2 non-proxy highs was directionally the single least-crowded segment in the dataset (`ANALYSIS_speed_lag_2026-11-13.md` §3 showed market-wide lead-2 takers earned `+3.9%` on `$350k` volume).
2. Our own `L1` check disproved Argon `F2` in the evening window: `temperature_v1` genuinely beats the old `143`-member ensemble by `-0.128` nats standalone and `-0.040` nats inside an identical `w = 0.5` `ask/2` log-pool (`17/23` to `19/23` dates).
3. Over only `23` correlated synoptic dates (`4` `val` dates happened to lose `-14.4%`), realized binary win rates have standard errors of `+-4%` to `+-6%`, so `-2.9%` (`2a`) and `+2.5%` / `+4.3%` (`3c`/`3d`) have overlapping confidence bands (`-9.7%..+4.2%` vs `-6.5%..+16.2%`). Live `blend.py` also already deducted `taker_fee` inside its edge gate, unlike `research_edge_after_costs.py`.

**Why our conclusion still stands:** Even granting every point above, deploying `blend_min_local_hour = 15.0` when the supporting backtest coded `>= 18:00`, crowding out high-edge modal buckets via ascending iteration, spamming `orders` rejections on `0.784..0.90` and `[held]` markets, benchmarking `blend` against `max(B1, B2)` on retired `forecast` markets, and leaving `PREREGISTRATION.md` §7 unimplemented in `scorecard.py` are **deterministic software and experimental-design defects**, not small-sample noise. Fixing them in `CO-006` strictly dominates leaving `CO-005` or Argon's partial `CO-006` in place.

---

## Prioritized Change Orders (`P0–P3`)

1. **`P0` — Superseding `CO-006` (`change_orders/2026-11-16/Carbon/CO-006.diff`, Ready to Apply Now):**
   - Raise `blend_min_local_hour` `15.0 -> 18.0`; lower `blend_max_price` `0.90 -> 0.78`; sort `blend.evaluate_event()` candidates by descending `edge`; skip `[held]`/`[opposite held]`/`[event cap]` before `broker.reject()`; bump `blend_version = "2026-11-16-v2"`; make `blend.inputs(..., now=now_utc)` deterministic and log `_refresh_fail` in `__main__.py`; add 11 `blend_snaps` columns (`input_age_min`, `local_hour`, `hours_to_peak`, `tradable`, `edge_yes`, `edge_no`, `fee_rate`, `yes_ask_depth`, `no_ask_depth`, `model_name`, `blend_version`).
   - Implement `scoring.baselines_blend()`, `scoring.blend_evaluation()` (`S1a` log-loss, `S1b` multiclass Brier, `S2` vs `forecast`, `S3` per-share signal return), `fill_realism(strategy='blend')`, and `tools/scorecard.py` + `PREREGISTRATION.md` §7 integration.
   - Fix `dashboard.py` `_latest_books()` to include `blend_snaps` (`ts` timestamped) and fall back to fresh `market_meta.mid_yes` when `mm_ts` is `> 2h` newer than `book_ts` on `lead <= 0` positions.
2. **`P1` — 24-Hour Hermes Quota & Live Parity Check (Post-Apply Day 1):**
   - Read Open-Meteo commercial dashboard units 24h after applying `CO-006`; if `blend` exceeds `50%` of monthly plan rate, set `blend_refresh_min = 110` in `config.json`.
3. **`P2` — `CO-007` (`models/temperature_v2` Retrain, Deferred Until 30 Effective Dates or Next Offline Build):**
   - Fit `gbt_var` and `var_scale` strictly on out-of-fold (`oof`) residuals in `pipeline/temperature_model.py`; replace Gaussian `ndtr` in `predict.bucket_probs()` with a heavy-tailed **Student-`t` (`df ~ 5..7`)** CDF calibrated on OOS `val` multiclass Brier + log-loss; exclude `is_label_proxy` stations (`VHHH`, `RCSS`) from `export_temperature_model.py`; remove retired `bom_access_global`; gate non-favourite `YES` longshots (`ask < 0.25`) behind a higher edge threshold (`>= 0.12`) validated strictly on `market_split` (`train` -> `val` -> untouched `test`).
4. **`P3` — Disk Hygiene (`PC-15`, Before `2026-11-28`):**
   - Archive raw CLOB snapshots older than 14 days on Hermes before `/opt/data` reaches capacity.

---

## Uncertainties and What Would Change This Verdict

- **What would upgrade `blend` from skeptical paper trial to genuine alpha candidate:** Passing all gates at `60` effective dates in unchanged epoch `2026-11-16-v2` (`S1a` log-loss band `< 0`, `S1b` multiclass Brier `<= 0`, `S2 < 0`, `S3` per-share net return `95%` band `> 0`, and `scorecard.py` fillable P&L `> B2_blend`).
- **What would trigger immediate retirement of `blend`:** At `30` effective dates in `2026-11-16-v2`, either `S1a`'s `95%` band fails to clear zero or `S3` mean per-share net return after `fee_rate` is `<= 0` (`PREREGISTRATION.md` §7 futility gate).

---

## Cheapest Way to Verify the Key Claims Yourself

1. **Verify the `15:00 vs 18:00` and zero-fee backtest bugs on your Mac (`< 2` seconds):**
   ```bash
   grep -n "no fees\|hours_to_day_start <=" <records>/pipeline/research_edge_after_costs.py
   ```
2. **Verify `69.6%` (`100%` since Nov 3) `0.05` `fee_rate` on your Mac (`< 3` seconds):**
   ```bash
   <records>/.venv/bin/python -c "import duckdb; print(duckdb.sql(\"SELECT min(target_date_local), max(target_date_local), fee_rate, count(*) FROM '<records>/data/ml/market/blend_bucket_scan.parquet' WHERE target_var='tmax' AND lead_days=2 GROUP BY fee_rate\").df())"
   ```
3. **Verify Carbon `CO-006.diff` (`Rev B`) applies cleanly (`--fuzz=0`) and preserves `PREREGISTRATION.md` §§1–6 (`< 3` seconds):**
   ```bash
   patch -d /tmp/polybot_verify/post_co005 -p1 --fuzz=0 --dry-run < /home/user/workspace/projects/polybot-vN9yqcIiTqOfKgSOh16png/files/change_orders/2026-11-16/Carbon/CO-006.diff
   ```

---

## Rev B Post-Review Errata (`2026-11-16`, Following Claude Fable 5.2 Clean-Room Audit)

All confirmed code and prose defects flagged by Claude Fable 5.2 against `snapshots/polybot_code_2026-11-14 + CO-005.diff + Carbon CO-006.diff (Rev A)` have been verified against primary source files and corrected in `CO-006.diff (Rev B)` and `CO-006-carbon-execution-scoring-and-governance.md (Rev B)`:

1. **`PREREGISTRATION.md` truncation (`Rev A` hunk `@@ -1,96 +1,19 @@`):** Conceded and fixed (`@@ -94,3 +94,21 @@`; preserves all 96 lines of Title + §§1–6 and appends §7, `114` lines total).
2. **Premature `"approved by Chase"` strings:** Conceded and removed (`PREREGISTRATION.md` §7 now reads `proposed 2026-11-16; pending approval by Chase`).
3. **`PREREGISTRATION` §7 gate in `scoring.go_live()`:** Conceded and wired (`go_live` for `blend` now appends `("PREREGISTRATION §7 60-date evaluation PASS", bev["verdict"], bev["verdict"].startswith("PASS"))`, returning `ready = False` on 100 winning trades across 20 dates while §7 is `PENDING`).
4. **Fee-inclusive price ceiling (`0.78 -> 0.77`) and cap/cash checks (`blend.py`):** Conceded and fixed (`ask = 0.78` yielded `$3.978` cost + `$0.04376` fee = `$4.02176 > $4.00` cap and `-$0.04376` cash from `$3.978`; `blend_max_price` is now `0.77` (`$3.97216 <= $4.00`) and `blend.py` checks fee-inclusive `est_total` against `blend_max_event_usd`, `blend_max_open_usd`, and `db.cash_balance`, with `$0.00` P/L credit attributed to the price ceiling).
5. **Identifier, baseline, snapshot, quota, replay-bracket, and dashboard errata:**
   - Real config keys are `"mode": "paper"` and `blend_max_event_usd` (`"live"`, `"forecast_Order"`, `"follow_enabled"`, and `"blend_max_per_event"` were prose errors; `forecast`/`follow` retirement lives in SQLite `strategy_status`).
   - `CO-005`'s `15:00` local gate came implicitly from `m_lead == lead` via `tempmodel.PEAK_HOUR = 15`, not a pre-existing `blend_min_local_hour` config key.
   - Real epoch helper is `scoring.current_epoch_start(con, cfg, "blend")` (not `strategy_epoch_ts`); `dashboard.py` had no `_latest_books()` helper, and on Asian Nov 16 rows (`UTC+8`/`UTC+9`) `Mark` (`0.2%..100%`) was fresh while `Market now` (`43%..60%`, frozen at `lead = 1` in `blend_snaps`) was stale, so `dashboard.py` is dropped from `Rev B`.
   - `scoring.baselines_forecast()` filters `e.kind = 'highest' AND s.lead_days IN (1, 2)` (highs only, not `highest + lowest`; its remaining mismatches for `blend` are `lead = 1` + pre-18:00 + `Hong Kong`/`Taipei` inclusion, zero-fee maker `NO` mid, and ex-post `max(B1, B2)` selection in `scorecard.py`).
   - `strategy.evaluate_event` still writes `snapshots` rows while `forecast` is retired (`fill_realism` reading `blend_snaps` is clean decoupling, not a missing-row bug).
   - Chase's Open-Meteo plan is **Professional (`5M` calls/month, `~14%` used)**, not `1M` Standard, and changing `blend_refresh_min` alters `governance.strategy_sha("blend", cfg)` and resets `blend`'s epoch.
   - Replay gain of `CO-006` across `23` unseen dates spans **`+$20.55` (`optimistic fc2`: `+$12.15 -> +$32.70`, with `+$4.78` from the `18:00` gate)** to **`+$82.79` (`knowable fc3`: `-$50.09 -> +$32.70`, with `+$67.02` from the `18:00` gate)**, bracketing live `K=0` (`~54h–61h` before target peak).
   - `audit_2026-11-16/screenshot_audit_results.json` resides in the Perplexity Project File Repo (`audit_2026-11-16/screenshot_audit_results.json`, generated `2026-11-16` from 40 manually transcribed dashboard screenshot rows) and was not synced to `<records>`; it is now explicitly shared in-thread as secondary UI-screenshot evidence.

