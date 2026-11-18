# CO-003: scoring fixes (exact t, maker-fill capacity, B2 at the mid)

**Status:** proposed 2026-10-30, for Chase's approval. **Risk:** none to trading. Measurement only: no decision path,
no epoch, no config key touched. **Patch:** `CO-003.diff` (`patch -p1` from the directory that holds `polybot/`;
verified to apply cleanly to the 2026-10-30 export's `scoring.py` and `PREREGISTRATION.md`).
**Files:** `polybot/polybot/scoring.py`, `polybot/notes/PREREGISTRATION.md`. Both are locked by
`notes/SCORING_MANIFEST.json`, so applying this raises the integrity FLAG until the manifest is re-registered (step 4).

## Why

Three defects in the official scorecard, all verified on the 2026-10-30 export:

1. **Capacity can never pass under the passive broker.** `fill_realism` measures capacity as BUY prints of our
   outcome at ≤ our price within −10/+30 min of the fill timestamp. A maker fill is stamped at the cycle that matched
   it, and the prints that filled it are SELLs into our bid (or BUYs of the complement) up to 30 min earlier. So every
   maker fill scores print_size 0: the 2026-10-30 scorecard shows "capacity median 0.0x" on 108 confirmed fills, and
   criterion 8 is unreachable for as long as the bot is passive.
2. **`t_crit` is a Hill-type series that is wrong where it matters.** Exact two-sided 95% Student-t vs the expansion:
   df 1.5 → 6.02 vs 4.80 (−20%), df 2 → 4.30 vs 3.85 (−10%), df 3 → 3.18 vs 3.06 (−4%), df 1.06 → 11.13 vs 12.71
   (+14%). Criterion 3 uses df = effective dates − 1, so the band is narrowest exactly in the 2.5–4 effective-date range
   the strategies are entering now.
3. **B2 pays the NO ask while forecast fills at or below the mid.** forecast's "beats its baseline" row compares a
   strategy that rests bids at mid (since 2026-10-25) with a baseline that lifts the ask. B2's −19.2% is mostly spread
   (selling the favourite at the NO mid loses about 2% on the trade-free test), which makes the row easier to pass
   than the registration intended.

A fourth point needs no code: the dashboard, SUMMARY.md and the kill rule show roi ± 2·SE (forecast −32.9% to +12.9%),
the registered scorecard shows t × SE (−273.7% to +252.2%) for the same 53 trades. Both are correct under their own
definition; the registration now says so in one sentence so the two are not read as a contradiction.

## What changes

1. `scoring.t_crit(df)`: exact two-sided 95% quantile via the regularised incomplete beta (pure Python, no new
   dependency) with bisection; df clamped at 1. Checked against reference values at df 1, 1.06, 1.5, 2, 3, 4, 5, 10,
   19, 30, 100 (all within 0.002). Bands get wider at low effective dates, never narrower, except df < 1.5 where the
   old code returned the df = 1 value (12.71) for everything.
2. `scoring._maker_capacity(con, fill)`: for `fills.maker = 1`, print_size = qualifying tape volume over the order's
   resting window (`execution._qualifying(resting_order, posted, expiry)`, the same rule that filled it) and
   capacity = that volume / the order's size. Taker fills keep the old measurement. Unit check: 8 through + 14 at price
   with queue 10 → 12 qualifying, order of 5 → capacity 2.4; no resting order → (0, None).
3. `baselines_forecast`: B2 pays the NO mid when a NO bid exists (the ask otherwise); the 0.05–0.80 window applies to
   that price. B1 unchanged (it is the floor, not a strategy's baseline). B3 still pays the ask; noted in the
   registration as "revisit if follow ever has fills to compare".
4. `PREREGISTRATION.md`: header amended; §2 row 3 says t is the exact quantile and names the ± 2·SE band as the
   governance band; §2 row 8 defines capacity per fill type; §3 B2 at the mid. `SCORING_MANIFEST.json` is not in the
   patch: it is rewritten by the re-registration step below.

Not changed: `governance.py` (kill rule stays ± 2·SE, as registered on 2026-10-21), `tools/scorecard.py` (reads
`capacity_median` and the bands as before; not in the export, so not patched; if it formats `t_crit` itself, check it).

## Apply (Hermes agent, workdir /opt/data/polybot)

```bash
# no cron pause needed: nothing here runs inside the trading path. Do it between cycles anyway (xx:05 or xx:35).
# 1. keep the "before" scorecard
cp notes/scorecard/$(ls notes/scorecard | tail -1) cache/scratch/scorecard_pre_co003.md
# 2. apply from the directory that contains polybot/ (production: /opt/data)
cd /opt/data && patch -p1 --dry-run < CO-003.diff && patch -p1 < CO-003.diff && cd /opt/data/polybot
.venv/bin/python -m py_compile polybot/scoring.py
# 3. run the scorecard once: it must now show the integrity FLAG (2 files changed since registration)
.venv/bin/python tools/scorecard.py
# 4. re-register (needs Chase's approval, which this CO is): rewrites notes/SCORING_MANIFEST.json
.venv/bin/python -c "from polybot import config, scoring; print(scoring.register(config.load(), 'CO-003 (2026-10-30, approved by Chase): exact Student-t quantile; maker-fill capacity from the resting window; B2 at the NO mid. Thresholds unchanged.'))"
# 5. run it again: integrity OK
.venv/bin/python tools/scorecard.py
```

## Verify

- Step 3 output starts with the FLAG lines for `polybot/scoring.py` and `notes/PREREGISTRATION.md`; step 5 shows
  "scoring files and thresholds match the registration". Thresholds in the manifest (freeze_days 7, kill_min_trades
  100, min_verdict_dates 15, global_freeze_until 2026-12-20) are unchanged.
- forecast card: "capacity median" is a number > 0 (expected order of 1–3x on the Oct 27–29 fills); criterion 8 now
  reads a real value. Row 3 band: wider than before at the same n and effective dates (forecast at 2.05 eff. dates:
  t goes 12.71 → 11.13, so slightly narrower there; from ~2.5 eff. dates on, wider).
- B2 favourite NO: return per $ moves from about −19% toward the trade-free "sell the favourite at the NO mid" number
  (−2% to −5% on the same window); n may change by a few (window applied to the mid).
- Paste the before/after scorecard tables into the CHANGELOG entry.

## Rollback

`patch -R -p1 < CO-003.diff`, then re-register once more with a note, otherwise the FLAG stays up. No data to undo.

## CHANGELOG entry (paste into notes/CHANGELOG.md)

```
## 2026-10-30 — scoring: exact t, maker-fill capacity, B2 at the mid (CO-003, measurement only; re-registered)
Evidence: 2026-10-30 scorecard showed capacity median 0.0x on 108 confirmed maker fills (fill_realism counted BUY prints
around the match timestamp; maker fills are SELLs into our bid inside the resting window). t_crit's series expansion ran
20% / 10% / 4% low at df 1.5 / 2 / 3. B2 paid the NO ask while forecast rests at mid since 10-25 (-19.2% was mostly spread;
trade-free NO at the mid loses ~2%).
Changes: scoring.t_crit exact (incomplete beta); scoring._maker_capacity = qualifying tape volume over (posted, expiry] /
order size for maker=1 fills; B2 priced at the NO mid (ask only where no bid); PREREGISTRATION §2 rows 3 and 8, §3 B2
amended and the ±2·SE governance band named as distinct from the t band. Kill rule, thresholds, decision code untouched.
Manifest re-registered. Before/after scorecard: ...
```
