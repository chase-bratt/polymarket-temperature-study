# CO-005: new strategy `blend` (temperature model + market price); forecast and follow retired; freeze ended

**Status:** proposed 2026-11-14, approved in principle by Chase ("ignore the freeze, restart the date counter with a
new strategy, one full change order"). **Mode:** paper only, as before. No cap is raised.
**Deliver to Hermes:** this document, `CO-005.diff`, `temperature_v1.zip`.

| file | what |
|---|---|
| `CO-005.diff` | unified diff, 10 files, `patch -p1` from the directory that holds `polybot/` (production: `/opt/data`). Built against `polybot_code_2026-11-14.zip`; applies cleanly and reproduces the tested tree |
| `temperature_v1.zip` | the model: `model.json` (6 models × 300 trees + uncertainty forests, 2.6 MB), `clim.json`, `predict.py`, `test_vectors.json`, `README.md` |

## 1. What changes, in one paragraph

The bot stops trading its ensemble-versus-market strategy (`forecast`) and the dormant `follow`, and starts a new
strategy, `blend`. `blend` forecasts each station's daily high with a trained model (nine weather models corrected by
gradient-boosted trees, with a second forest for the uncertainty), blends that with the market's price, and buys a
bucket side at the ask when the blend says it is worth at least 8 cents more than it costs. It trades only markets two
days ahead, and only from 15:00 station time. Every evaluated bucket is logged whether traded or not, so the edge can
be scored against the market with exact timing.

## 2. Why (evidence; records folder `data/reports/`)

1. **The old strategy has no edge.** `forecast`, current epoch: 295 settled, 168 won against 162.7 implied by the
   prices paid and 205 expected by its pool, −$2.13. Its model lost to the market's price on 21 of 21 dates one day ahead.
   `follow`: 0 settled in two weeks.
2. **A better temperature model exists.** Trained on archived forecasts of nine models (2024-02 →) against airport
   observations (2000 →). Daily-high miss on a sealed test window (2026-03 → 2026-10, opened once):

   | | raw 9-model mean | tuned model |
   |---|---|---|
   | 1 day ahead | 1.52 °C | 1.13 °C |
   | 2 days ahead | 1.65 °C | 1.30 °C |

3. **Blended with the price, it beats the market two days ahead.** 23 unseen market dates, timing restricted to what
   was knowable: blend log-loss minus market **−0.04 (−0.06 to −0.02)**, better on 19–20 of 23 dates. One day ahead the
   blend is worse than the market (+0.03 to +0.04), which is why one day ahead is scored but not traded.
4. **It is not proven to pay the spread.** Trading replays two days ahead: from −3% (strict timing) to +7% (after
   15:00), every interval including zero. `blend` is a paper test of the edge in point 3. The stopping rule in §7 was
   written before any live result.
5. **Breadth, not speed, was the gap.** The market moves before the weather centres publish the ensembles the old
   bot waited for (`arrival_alignment.md`); adding seven models and forecast weather variables is what moved the score.

## 3. The strategy rule (exactly what the code does)

For every daily-high event whose station-local date is 1 or 2 days ahead (Hong Kong and Taipei excluded: they do not
resolve on the airport METAR):

1. **Model lead** L = whole days, rounded up, from now to 15:00 station time on the target day (clamped 1–3). The
   lead-L model was trained on forecasts exactly L days old, so rounding up means the live forecast is never older than
   the model expects.
2. **Inputs** for the station (§4), refreshed at most every 55 minutes → `mu`, `sigma` in °C from `tm_predict`.
3. **Model probabilities** per bucket: normal(mu, sigma) over the bucket's range with the market's half-up rounding;
   °F markets are converted first.
4. **Market probabilities** per bucket: YES mid where a bid exists, ask/2 where none does, normalised over the event.
   Fewer than half the buckets bid = thin book: scored without a blend, never traded.
5. **Blend:** log pool, p ∝ model^0.5 × market^0.5, renormalised.
6. **Log** one `blend_snaps` row per bucket (leads 1 and 2).
7. **Trade** only if: station-local lead is 2, model lead is 2 (i.e. from 15:00 station time two days before), inputs
   are under 3 hours old, the strategy is not retired. For each side (YES at the YES ask, NO at the NO ask):
   buy when `blend probability − ask − fee ≥ 0.08`, ask within 0.05–0.90, probability ≥ 0.05. Taker order at the
   displayed ask through the existing `PaperBroker` (walks the book, respects the 5-share minimum). $2 per order (up
   to the 5-share minimum if that is under $4), $4 per event, $200 open. One position per market and side; never both
   sides of one bucket.

Fills are takers at the displayed ask. The existing scorecard already classifies each such fill as confirmed by a real
print, persisted, or contradicted, and reports "fillable P/L" separately: that row, not the paper row, is the judge.

## 4. The feature recipe (`polybot/tempmodel.py`)

This is the part that must not drift. Every rule mirrors the training table (`pipeline/ml_temperature_hist.py` in the
records folder). **Verified:** the module, run against Open-Meteo's public hosts, rebuilt 144 recent training rows
(8 stations × 3 dates × highs and lows × leads 1–3); every forecast-side input matched to rounding.

Three Open-Meteo calls per station per refresh, all with `timezone=GMT` (never local time: Open-Meteo applies one fixed
offset per response and ignores daylight saving; hours are assigned to station-local days with the real time zone):

| call | host (keyed customer host when the key is set) | gives |
|---|---|---|
| A | previous-runs API, `models=` the 9 ids, `temperature_2m` + `_previous_day1..3`, past 37 d + 4 d ahead | each model's newest forecast and the forecasts shown 1, 2, 3 days earlier |
| B | previous-runs API, GFS + ECMWF, dewpoint / cloud / wind / pressure, newest + `_previous_day1..2` | forecast weather for the target day |
| C | forecast API (best match), 8 variables, past 37 d | the models' newest-run "analysis" of each past day |

Plus the bot's own `observations` table: routine METARs only, T-group tenths, day by true observation time; a day counts
when it has ≥ 20 reports and a later local day has a report.

| inputs | rule |
|---|---|
| `fc_<model>_c`, `fc_mean_c`, `n_models`, `fc_sd_c`, `fc_spread_c`, `model_diff_c`, `d_<model>_c` | daily max of each model's newest forecast over the station-local target day (≥ 20 hourly values); a model more than 12 °C from the models' median (≥ 3 models) is dropped as a data fault |
| `gfs_*`, `ecmwf_*` (dewpoint, cloud, wind, pressure) | daily mean of the newest forecast for the target day; leads 1–2 only |
| `rev_mean_c`, `rev_ecmwf_c`, `rev_gfs_c` | newest forecast minus the forecast one day older, same target day |
| `err_mean_30d_c`, `err_sd_30d_c`, `err_mean_7d_c`, `err_last_c`, `err_ecmwf_30d_c`, `err_gfs_30d_c` | station observation minus the lead-L forecast (the `_previous_dayL` columns), over target days in [target − L − 30, target − L − 1] |
| `lag_anom_c`, `an_lag_anom_c`, `stn_minus_an_lag_c`, `an_*_lag_*`, `an_bias_30d_c` | the day target − L − 1: station extreme, the best-match analysis of that day, and station minus analysis over the same 30-day window |
| `clim_anom_c` | forecast mean minus `clim.json` (normal for that day of year from earlier years; regenerate each January) |
| `lat`, `lon`, `doy_sin`, `doy_cos`, `station_code` | static; station codes are in `model.json` |

A missing input is `None`. Never fill a gap with 0: the trees were trained with missing values and route them.

## 5. Files (all in `CO-005.diff`)

| file | change |
|---|---|
| `polybot/polybot/tempmodel.py` | NEW. Fetch, reduce to station-local daily values, build the model's inputs, choose the model lead |
| `polybot/polybot/tm_predict.py` | NEW. The model runtime: pure Python, no dependencies (identical to the package's `predict.py`) |
| `polybot/polybot/blend.py` | NEW. The strategy: tables `blend_snaps` and `blend_inputs`, market probabilities, log pool, trading rule |
| `polybot/polybot/__main__.py` | the scan runs `blend` after `forecast`, inside try/except; counts added to the run note |
| `polybot/polybot/config.py`, `polybot/config.json` | `blend_*` settings; `global_freeze_until` set to 2026-11-14 (freeze ended) |
| `polybot/polybot/governance.py` | `STRATEGIES` gains `blend` (own epochs, freeze clock and kill rule) |
| `polybot/polybot/review.py` | review report lists `blend` |
| `polybot/tools/scorecard.py` | go-live checklist for `blend` (baseline = better of favourite YES / NO in its window); freeze line reads "no freeze in force" |
| `polybot/notes/PREREGISTRATION.md` | §6: the change, blend's criterion and the stopping rule |

`governance.py`, `tools/scorecard.py`, `PREREGISTRATION.md` and `global_freeze_until` are locked by the manifest, so the
scorecard flags them until step 8 re-registers.

Not touched: `strategy.py`, `follow.py`, `execution.py`, `broker.py`, `scoring.py`, capture, settlement, the dashboard
plugin. Nothing is deleted. The retired `forecast` keeps writing its snapshots, as a comparison.

## 6. Apply (Hermes, workdir `/opt/data/polybot`)

Disk first: this order adds two small tables and writes about 1,100 `blend_snaps` rows per cycle (≈ 3 MB/day). It does
**not** need a copy of the ledger; nothing existing is altered. Do not make a full backup if the disk cannot hold it.

```bash
# 1. pause the three cron jobs (cycle, settle-fast, runwatch) and wait for any running cycle to finish

# 2. install the model and prove it is the tested one
mkdir -p models && unzip -o /path/to/temperature_v1.zip -d models/
(cd models/temperature_v1 && ../../.venv/bin/python predict.py --selftest)      # must print PASS
(cd models/temperature_v1 && ../../.venv/bin/python predict.py --info)

# 3. apply the diff from the directory that contains polybot/
cd /opt/data && patch -p1 --dry-run < /path/to/CO-005.diff && patch -p1 < /path/to/CO-005.diff && cd /opt/data/polybot
.venv/bin/python -m py_compile polybot/*.py tools/scorecard.py

# 4. retire the old strategies and pull their resting bids (open positions settle normally)
.venv/bin/python - <<'PY'
from polybot import db, execution, governance
con = db.connect(); governance._ensure(con); execution.ensure(con)
for st, why in (("forecast", "replaced by blend (CO-005): no edge after 295 settled; model lost to the market"),
                ("follow", "replaced by blend (CO-005): dormant, 0 settled since 2026-10-30")):
    n = execution.cancel_open(con, st, "strategy retired (CO-005)")
    con.execute("INSERT OR REPLACE INTO strategy_status VALUES (?,?,?,?)", (st, "retired", db.now(), why))
    print(st, "retired;", n, "resting bids pulled")
con.commit()
PY

# 5. one-station check of the live inputs (uses the keyed hosts; prints no key)
.venv/bin/python - <<'PY'
import datetime as dt
from zoneinfo import ZoneInfo
from polybot import config, db, blend, tempmodel, tm_predict, stations
cfg = config.load(); con = db.connect(); model, clim = blend.package(cfg)
icao = "klga"; lat, lon, tz = stations.STATIONS[icao]
target = (dt.datetime.now(ZoneInfo(tz)).date() + dt.timedelta(days=2)).isoformat()
inp = tempmodel.gather(lat, lon, tz)
obs = tempmodel.station_obs_daily(con, icao, tz, "2026-10-01")
L = tempmodel.model_lead(target, tz)
f, fc_mean, spread = tempmodel.build(inp, obs, clim[icao], {"code": model["station_codes"][icao], "lat": lat, "lon": lon}, target, "tmax", L)
mu, sigma = tm_predict.forecast(model, "tmax", L, f, fc_mean, spread)
print(f"{icao} {target}: model lead {L}, {f['n_models']} models, mean {fc_mean:.1f} C -> {mu:.1f} C +- {sigma:.1f}; "
      f"complete obs days {sum(o['complete'] for o in obs.values())}; inputs missing: {sorted(k for k, v in f.items() if v is None)}")
PY

# 6. one manual cycle
./cycle.sh
.venv/bin/python -m polybot governance

# 7. look at what blend wrote
sqlite3 data/polybot.sqlite "SELECT lead_days, model_lead, count(*), count(p_pool), round(avg(sigma_c),2) FROM blend_snaps GROUP BY 1,2"
sqlite3 data/polybot.sqlite "SELECT status, count(*) FROM orders WHERE strategy='blend' GROUP BY 1"

# 8. re-register the scoring manifest (Chase's approval is this order), then confirm integrity is clean
.venv/bin/python tools/scorecard.py --register "CO-005 (2026-11-14, approved by Chase): blend added to governance and the scorecard; forecast and follow retired; freeze ended; PREREGISTRATION section 6. Thresholds unchanged."
.venv/bin/python tools/scorecard.py | head -20

# 9. resume the cron jobs
```

## 7. Verify (all must hold before resuming the crons)

- Step 2 prints `PASS: 360 vectors`. Anything else: stop, the package is not the tested model.
- Step 5 prints 8 or 9 models, a mean and a prediction within a few degrees of each other, sigma between about 0.6 and
  2.5 °C, at least 20 complete observation days. Expected missing inputs: `d_bom_c` (that model's archive ended in 2025)
  and nothing else for a station with a month of METARs. More than a handful missing: stop and report which.
- Step 6: the cycle log has `(blend)` blocks; no `blend failed` lines; `governance` shows `forecast` and `follow`
  RETIRED and a `blend` epoch starting at the cycle time with no second epoch after it.
- Step 7: rows for `lead_days` 1 and 2; `count(p_pool)` equals `count(*)` except on thin books. Orders, if any, only
  from events two station-local days ahead whose station time was past 15:00 (`lead_days = 2 AND model_lead = 2`).
  Zero orders on the first cycle is normal: it depends on the hour.
- Step 8: the scorecard's Integrity section reads "scoring files and thresholds match the registration" and "No freeze
  in force"; the first go-live block is `blend`.
- No trade on Hong Kong or Taipei, none on lows, none one day ahead.
- Quota: each station refresh is three calls on the paid previous-runs / forecast hosts, at most hourly, about 3,600
  calls a day. Report the measured units for the first 24 hours. The paid plan must stay active: the previous-runs
  host is what keeps live inputs identical to the training data.
- Dashboard: confirm a `blend` card appears and retired strategies are hidden as before. If the card is missing, say
  so; the dashboard plugin was not changed by this order.

After the first week, report: cycles with blend errors, median input age, positions opened, and the trade-free score
from `blend_snaps` (blend vs market log-loss by lead) once markets have resolved.

**Reviewer prompts:** update the daily and weekly review prompts: the active strategy is `blend`; `forecast` and
`follow` are retired; the freeze has ended; the stopping rule below replaces the freeze text. Reviewers still change
no parameter.

### The stopping rule (also in PREREGISTRATION §6)

After 20 effective event dates: if blend's fillable P/L band is not above zero, or its trade-free score (blend
log-loss minus market, two days ahead) is not below zero on a clear majority of dates, retire `blend`. No parameter
(pool weight, edge threshold, hours, caps) is tuned on blend's own live results before then.

## 8. Rollback

```bash
cd /opt/data && patch -R -p1 < /path/to/CO-005.diff && cd /opt/data/polybot
sqlite3 data/polybot.sqlite "DELETE FROM strategy_status WHERE strategy IN ('forecast','follow')"   # only to un-retire them
.venv/bin/python tools/scorecard.py --register "CO-005 rolled back"
```

`blend_snaps`, `blend_inputs`, blend's orders, fills and positions stay in the ledger as history (open blend positions
settle normally; nothing reads the two tables after the rollback). The `models/` folder can stay.

## 9. Not in this order

- Lows, one-day-ahead trading, passive (maker) bids for blend, sizing by Kelly: no evidence yet.
- Retraining. `temperature_v1` is fitted through 2026-10-09 so that the market-period tests stay honest. Retrain on the
  laptop (`pipeline/export_temperature_model.py --through <date>`) and ship a new zip when wanted; the bot only needs
  `blend_model` pointed at the new folder and a `blend_version` bump.
- The raw nine-model capture task (`hermes_live_multimodel_log.md`): still useful for retraining with exact timestamps,
  but no longer required to score the edge, since `blend_snaps` records every decision input's result.
- Low-cloud forecasts: the column was empty in the download, so the model does not use it. Worth re-fetching.

## 10. CHANGELOG entry (paste into notes/CHANGELOG.md)

```
## 2026-11-14 — CO-005: blend strategy; forecast and follow retired; freeze ended (approved by Chase)
Evidence (records folder data/reports): forecast 295 settled / 168 won vs 162.7 price-implied vs 205 pool-implied, -$2.13;
its model lost to the market on 21/21 dates at lead 1. New temperature model (9 archived models + trees, variance forest):
daily-high RMSE 1.13 C (lead 1) / 1.30 C (lead 2) on a sealed test window vs 1.52 / 1.65 for the raw mean. Blended with
the price (log pool w 0.5), knowable timing, 23 unseen dates: -0.04 nats vs the market at lead 2 (-0.06 to -0.02, 19-20
dates); worse than the market at lead 1. Trading replays not distinguishable from zero (-3% to +7%).
Changes: polybot/tempmodel.py (live inputs; rebuilt 144 training rows to rounding), tm_predict.py (pure-Python runtime,
self-test PASS on 360 vectors), blend.py (highs, lead 2, from 15:00 station time, taker at ask, edge >= 0.08, $2/$4/$200,
HK and Taipei excluded, every bucket logged in blend_snaps), scan wiring, blend_* config, governance STRATEGIES, scorecard
go-live block for blend, PREREGISTRATION section 6 (stopping rule at 20 effective dates). forecast and follow retired in
strategy_status; resting bids pulled; open positions settle normally. global_freeze_until -> 2026-11-14. Manifest
re-registered. Model package models/temperature_v1 (fitted through 2026-10-09).
First cycle: ...
```
