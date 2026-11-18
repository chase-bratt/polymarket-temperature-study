# Audit of CO-005 (`blend`), the temperature model `temperature_v1` and the 11-13 analyses

2026-11-16. Method: start from "CO-005 is flawed" and try to prove it. Read: `CO-005-blend-strategy.md`, `CO-005.diff`
(all 10 files), `ANALYSIS_review_2026-11-13.md`, `ANALYSIS_speed_lag_2026-11-13.md` and the three session transcripts
(the last ends 2026-11-13 14:01Z). Not available: the Mac did not respond on 2026-11-16, so `temperature_v1.zip`, the
training tables, `data/reports/edge_after_costs.md`, `arrival_alignment.md` and `pipeline/*.py` were not inspected.
No transcript covers the training session (11-13 14:00Z to 11-14), so every model metric below is the CO's own
statement, not a number I reproduced.

## Verdict

**Partly right that it is flawed, medium confidence.** The code that puts the model into the bot looks careful
and mostly correct. The flaws are in the evidence and the test. The evidence that blend makes money is weaker than the
CO says. The test the CO sets up cannot tell a working strategy from a broken one. One reason the CO gives for its
timing rule is false. The paid-data usage is probably several times higher than the CO estimates.

What holds up:

- Retiring `forecast` was justified. In its current epoch it settled 295 trades and won 168, against 162.7 implied
  by the prices it paid and 205 expected by its own pool, for −$2.13. One day ahead its model lost to the market price
  on 21 of 21 dates (CO-005 §2.1; review §2).
- The code handles daylight saving and missing values correctly, includes a 360-vector self-test, excludes HK and
  Taipei (whose METAR labels did not match resolutions), and stays paper-only.

## Findings, most load-bearing first

### F1. The stopping rule cannot answer its own question. High confidence

The CO's §7 / PREREGISTRATION §6 rule retires blend after 20 effective dates unless blend's fillable P/L band is above
zero. I simulated that test with $2 trades bought at the ask (prices spread evenly from 0.08 to 0.85), 3 to 8 trades a
date, 95% bands over dates, with and without a shared shock per date. The script is in the appendix. These are my
assumptions, not measured inputs.

| true edge per share | chance the band clears zero after 20 dates |
|---|---|
| 0¢ | 1–3% |
| 4¢ (about the +7% top of the CO's replay range) | 6–16% |
| 8¢ | 17–48% |

- If the replay's best case were true, the rule would still retire blend about 85–90% of the time.
- The second clause compares blend's log-loss with the market's. Blend is a log pool that contains the market price
  (w = 0.5), and a pool usually beats its own ingredient whenever the model carries any independent information. So
  that clause is close to guaranteed to pass and says little about whether blend can pay the spread.
- Net effect: after 20 dates the rule will almost surely say "retire", and that outcome reflects the rule's design,
  not the strategy.

### F2. The model's added value over what the bot already had is not shown. Medium confidence

- **Market score.** Before any machine learning, the old ensemble pool already beat the market on highs two days ahead
  by −0.034 ± 0.020 nats, better on 15 of 20 dates (review §2 table). The new blend scores −0.04 (−0.06 to −0.02),
  better on 19–20 of 23 dates (CO-005 §2.3).
  - The gap between the two is about 0.006 nats, well inside either interval.
  - The two numbers come from different date sets and timing rules, so they cannot be compared directly at all.
- **Miss size (RMSE).** The CO's table compares the tuned model (1.30 °C at lead 2) with a raw 9-model mean
  (1.65 °C). The live 140-member ensemble the bot already ran was about 1.25 °C at lead 2 (review §3). The windows
  differ (sealed test 2026-03 → 10 vs the live period), so this does not prove the new model is worse. It does mean the
  CO's headline compares against a weak baseline.
- **Validation ladder.** On validation, boosted trees tied a one-line linear correction (EMOS) on highs: 1.34 vs
  1.34 °C (transcript 1, ladder table). That ladder used only 2 models; the 9-model version was not in the transcripts.
- **What would settle it:** the same 23 dates, scored with identical timing, comparing the new blend against the old
  pool, with a paired band (task L1 in CO-006).

### F3. The "unseen" dates are unseen by the model, not by the strategy design. Medium-high confidence

- The training split keeps market dates out of training: the market holdout starts 2026-10-17 (transcript 3, split
  config). The strategy choices, however, were made by looking at those same dates:
  - **Highs at lead 2:** the only winning cell out of 4 in the 11-13 review table (20 dates, which overlap the 23).
  - **From 15:00 station time:** matches the "+7% (after 15:00)" replay variant. The strict-timing variant was −3%.
  - **Pool weight:** "0.25–0.75 scored alike" (`config.py` comment), so it was inspected on these dates.
  - **The 0.08 threshold:** where it came from is unrecorded in what I read.
- Choosing the best cell, timing and settings from several variants tested on the same 23 dates biases the reported
  −0.04 and the replay range in blend's favour. I cannot size that bias without the variant count (task L1).
- Cities on the same date share weather, so the effective sample is about 23, not 1,000+ events.

### F4. The forecast-freshness reason in the CO is wrong. High confidence on direction, unknown size

- The CO says rounding the model lead up "means the live forecast is never older than the model expects" (§3.1, and
  the `model_lead` docstring). The facts:
  - **Training:** the lead-2 model learned from Open-Meteo `_previous_day2` values, defined as "the value that was
    predicted 48 hours before valid time" ([Open-Meteo Previous Runs API docs](https://open-meteo.com/en/docs/previous-runs-api),
    page undated, read 2026-11-16).
  - **Live:** blend uses the newest run (K = 0, `tempmodel.build`: `k0 = lead if asof else 0`). A run is always
    initialised before it is published, and the bot measured Open-Meteo publishing GFS 5.9 h and ECMWF 7.7 h after
    run start (speed-lag §6).
  - **So at 15:00 two days ahead,** the newest run is more than 48 h old at the target peak, by the publication delay
    plus the time since publication. The live input is older than the trained lead, not fresher.
- **Inference:** sigma is probably slightly too narrow live. The size is unknown.
- **The CO's "verified" check does not cover live trading.** It "rebuilt 144 training rows to rounding" with
  `asof=True` (K = lead). That proves the archive path matches the training table. It never exercises the K = 0 path
  the bot actually trades on.
- **Fix:** CO-006 corrects the docstring. Tasks L2 and L3 measure the effect.

### F5. Buying at the ask is where the bot's own evidence says it loses. Medium confidence

- At the bot's speed, crossing the spread two days ahead lost 3–6¢ a share in all four cells, with every interval below
  zero, and the spread at lead 2 is 5–8¢ (speed-lag §4–5).
- The 0.08 edge threshold exceeds that spread, which is the right shape.
- But the trades fire exactly where model and market disagree most. Those are the cases where the trees are
  extrapolating or inputs are missing, and where a winner's-curse shortfall is largest.
- **Steelman:** takers as a class earned +3.9% (+1.9% to +6.4%) two days ahead on $350k (speed-lag §3), so taking at
  lead 2 can pay for someone with better information. Whether blend is that someone is exactly what F1's test fails
  to measure.

### F6. Paid-data use is probably 3–7× the CO's estimate. Medium confidence

- **The CO's count:** "three calls per station per refresh … about 3,600 calls a day" (§7), which implies 50 stations
  refreshed hourly.
- **How Open-Meteo counts:** a request with more than 10 variables or more than 2 weeks is billed as several calls
  ("2 weeks … 15 weather variables … 1.5 API calls, while 4 weeks equals 3.0"), per
  [Open-Meteo pricing](https://open-meteo.com/en/pricing) (read 2026-11-16).
- **Blend's requests:** each asks for 41 days. Call A asks for 4 temperature series × 9 models, call B for 12 series
  × 2 models, call C for 8 variables.
- **My estimate:** about 9 to 20 billed calls per refresh, depending on whether each model counts as a separate
  variable (the page does not say). That is about 330k to 730k calls a month for blend alone, before `forecast`'s
  continued fetches. The Standard plan includes 1M a month; the free tier is 300k.
- **Not measured.** The CO's own step "report the measured units for the first 24 hours" is the check.
- Global models update every 6 h, so a 55-minute refresh mostly re-downloads unchanged data.

### F7. The log cannot fully reconstruct the decisions it is meant to score. High confidence (code)

- `blend_snaps` stores no input age, no "tradable" flag, no per-side edge, no fee rate and no model or strategy
  version. The registered "exact timing" score therefore cannot apply the CO's own trade gate (inputs ≤ 180 min old)
  after the fact.
- A failed input refresh is swallowed silently (`except Exception:` in `blend.inputs`), so stale-input scans look the
  same as fresh ones.
- CO-006 fixes both.

### F8. Blend's do-nothing baseline may be computed on the wrong set of markets. Low confidence (unverified)

- `scorecard.py` takes blend's baseline from `scoring.baselines_forecast(con, b_since)`. That function's market set is
  defined in `scoring.py`, which is not in the diff. If it includes lows and lead 1, the baseline is not blend's
  universe (highs, lead 2, after 15:00, excluding HK and Taipei).
- Check: task L5.

### F9. Smaller data inconsistencies. Low impact

- **Adverse selection is reported twice with different numbers.** The review has filled bids +1.4¢ and unfilled
  +7.3¢ per share (review §1). The pipeline has filled −0.3¢ and unfilled +3.15¢ (transcript 3). The direction agrees;
  the sizes do not. Probably different windows, but I could not confirm.
- 47,285 trade prints sit inside the documented 11-02 → 11-04 outage (probably backfilled).
- 680 forecast scans have zero ensemble members.
- Not blockers for CO-005, but any replay that uses those windows should say so.

### Correction to my own earlier advice

On 2026-10-30 I suggested lows and lead 2 were less crowded. The 11-13 corrected scoring (no-bid buckets at ask/2)
shows the market beats the pool on lows at both leads: 3 of 21 and 8 of 20 dates (review §2). The lows part of that
advice was wrong. The lead-2 highs part is what CO-005 now relies on.

## What would change the verdict

- **L1** shows the new blend beating the old pool on identical events, with a paired band below zero → F2 is cleared.
- **L1** shows the 23-date result survives the number of variants tried (for example, only 1–2 variants were scored)
  → F3 weakens to low.
- **L2** shows the lead-2 model's coverage barely changes with staler inputs → F4 is cosmetic.
- **Measured Open-Meteo units** under 3× the CO's estimate → F6 is withdrawn.

## Cheapest way to check the key claim yourself

- **F1:** run the appendix script with your own guess of trades per date. It takes about 30 seconds.
- **F7:** run this on Hermes. The output lists exactly the 16 columns in `ensure()`, with no input age or tradable flag.

  ```bash
  sqlite3 /opt/data/polybot/data/polybot.sqlite "PRAGMA table_info(blend_snaps)"
  ```

## Appendix: power simulation (F1)

```python
import numpy as np
rng = np.random.default_rng(1)
def sim(e, trades_per_date, dates=20, reps=4000, rho=0.0):
    hits = 0
    for _ in range(reps):
        per = []
        for d in range(dates):
            a = rng.uniform(0.08, 0.85, trades_per_date)          # ask paid
            z = rng.normal() * rho                                  # shared weather shock for the date
            win = rng.uniform(size=trades_per_date) < np.clip(a + e + z * 0.15, 0, 1)
            per.append(np.where(win, 1 / a - 1, -1.0).mean())       # return per $ staked
        per = np.array(per); m = per.mean(); se = per.std(ddof=1) / np.sqrt(dates)
        hits += (m - 1.96 * se) > 0
    return hits / reps
for e in (0.0, 0.02, 0.04, 0.08):
    print(e, [round(sim(e, k), 3) for k in (3, 8)], "with date shock", round(sim(e, 8, rho=1.0), 3))
# run 2026-11-16: 0.0 [0.011, 0.02] 0.027 | 0.02 [0.029, 0.067] 0.058 | 0.04 [0.059, 0.155] 0.11 | 0.08 [0.172, 0.48] 0.326
```
