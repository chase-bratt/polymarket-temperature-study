# CO-006: blend's evaluation replaced; decision logging completed; freshness claim corrected

**Status:** proposed 2026-11-16, needs Chase's approval (it amends PREREGISTRATION). **Risk:** low. It changes no
trading parameter, no strategy version and no epoch: paper only, as before.
**Basis:** `AUDIT_CO-005_blend_2026-11-16.md` (findings F1–F8). **Patch:** `CO-006.diff`, applies on top of CO-005
with `patch -p1` from the directory that holds `polybot/` (production `/opt/data`). Tested against the CO-005 files
rebuilt from `CO-005.diff`. `__main__.py` and `PREREGISTRATION.md` were tested on fragments only, because their full
pre-CO-005 text was not available, so use `--dry-run` first.

**Timing:** apply before blend has many settled dates. The new rule only counts as pre-registered if nobody has looked
at blend's results first. Do not open blend's live results before step 5.

## Files

| file | change |
|---|---|
| `polybot/polybot/blend.py` | `blend_snaps` gains 8 columns, added in place (`ALTER TABLE`; old rows keep NULL): `input_age_min`, `hours_to_peak`, `tradable`, `edge_yes`, `edge_no`, `fee_rate`, `model_name`, `blend_version`. Edges are computed for both sides on every row, ordered or not. A failed input refresh is recorded instead of swallowed. The decision rule is unchanged |
| `polybot/polybot/__main__.py` | prints one line per scan listing stations whose input refresh failed |
| `polybot/polybot/tempmodel.py` | docstring only: `model_lead` claimed live inputs are never older than training; they are older (F4) |
| `polybot/notes/PREREGISTRATION.md` | new §7 replaces §6's stopping rule with a three-score evaluation at 30 and 60 effective dates (F1–F3) |

Not touched: every `blend_*` parameter, `blend_version`, `scoring.py`, `scorecard.py`, `governance.py`, the broker, the
model package.

## Why

- **F1.** The current rule (20 dates, fillable P/L band) passes a 4¢-a-share strategy only about 10–15% of the time.
  Its log-loss clause compares a pool with its own ingredient.
- **F2.** Nothing currently tests whether the new model beats the retired pool, which already scored −0.034 at
  lead 2.
- **F7.** Without input age, the tradable flag and edge in each row, the "exact timing" score cannot be reproduced.

## What changes in the evaluation (full text in the diff, PREREGISTRATION §7)

| score | what it asks | decides |
|---|---|---|
| S1 | blend log-loss − market log-loss, at the first moment the rule could trade | must be below zero |
| S2 | blend log-loss − retired `forecast` pool log-loss, same markets | is the new model worth it |
| S3 | every signal with edge ≥ 0.08: outcome − ask − fee, per share, ordered or not | does it pay the spread |

- **At 30 effective dates:** retire if S1's band is not wholly below zero, or S3 ≤ 0.
- **At 60 effective dates:** pass only if S1's band is below zero, S3's band is above zero, and S2 < 0.

Expected behaviour (simulation, stated assumptions, see audit):

| true edge | outcome |
|---|---|
| none | retired at 30 dates about half the time; passes at 60 about 3% |
| 4¢ a share | passes at 60 about 40% |
| 6¢ a share | passes at 60 about 80% |

The kill rule and §2 go-live criterion are unchanged.

## Apply (Hermes, workdir `/opt/data/polybot`)

```bash
# 1. pause the cycle, settle-fast and runwatch crons; wait for a running cycle to finish
# 2. patch
cd /opt/data && patch -p1 --dry-run < /path/to/CO-006.diff && patch -p1 < /path/to/CO-006.diff && cd /opt/data/polybot
.venv/bin/python -m py_compile polybot/*.py
# 3. migrate the table (idempotent) and confirm the columns
.venv/bin/python -c "from polybot import db, blend; c=db.connect(); blend.ensure(c); print([r[1] for r in c.execute('PRAGMA table_info(blend_snaps)')])"
# 4. one manual cycle
./cycle.sh
# 5. re-register the manifest (PREREGISTRATION is locked)
.venv/bin/python tools/scorecard.py --register "CO-006 (2026-11-16, approved by Chase): blend evaluation replaced (PREREGISTRATION section 7); blend_snaps decision columns; no parameter or version change."
.venv/bin/python tools/scorecard.py | head -20
# 6. resume the crons
```

## Verify

- **Step 3:** prints 24 columns, ending `input_age_min … blend_version`. Running it twice is harmless.
- **Step 4:** no `blend failed` lines. Any `input refresh failed for N station(s)` line names the stations. New rows
  have `model_name = temperature_v1` and `blend_version = 2026-11-14-v1`.
- **Check the new rows with:**

  ```bash
  sqlite3 data/polybot.sqlite "SELECT tradable, count(*), round(avg(input_age_min)), round(min(hours_to_peak),1), round(max(hours_to_peak),1) FROM blend_snaps WHERE input_age_min IS NOT NULL GROUP BY 1"
  ```

  - `tradable = 1` rows should only have `hours_to_peak` ≤ 48.
- **Consistency of every order with its logged edge:** every blend order placed after the patch must have a
  `blend_snaps` row from the same scan and market whose `edge_yes` or `edge_no` is ≥ 0.08.
- **Step 5:** Integrity reads "scoring files and thresholds match the registration".
- **Quota (F6):** report Open-Meteo's billed units for the first 24 h after CO-005/CO-006. If blend alone uses more
  than half the plan, raise `blend_refresh_min` from 55 to 110. That is a quota fix with no decision effect, because
  global runs arrive every 6 h. Record it in the CHANGELOG.

## Laptop tasks (records folder; not in the diff, results go in `data/reports/`)

- **L1, same-events comparison.** On the 23 dates, use the exact snapshots `edge_after_costs.md` used.
  - Report the new blend minus the old pool, with a paired band over dates.
  - List every variant scored on these dates: kind × lead × timing × w × threshold. Report how many were tried before
    lead-2 highs after 15:00 was chosen.
- **L2, staleness bracket.** On the sealed test window, feed the lead-2 model its `_previous_day3` inputs instead of
  `_previous_day2` (24 h staler, an upper bound on live staleness). Report RMSE, log-loss and 80% coverage for both. If
  coverage drops below 75%, live sigma is too narrow; that becomes a later retrain, not a live tweak.
- **L3, live-path parity.** For 2 weeks of decision snapshots, rebuild each row after the fact with `asof=True` and
  compare with the live row: `fc_mean_c` difference, and missing-input rate live vs the training table (esp. the
  `err_*` inputs, which need ≥ 20 METARs a day from the bot's own table).
- **L4, scorer.** Write the deterministic S1–S3 script as specified in PREREGISTRATION §7, with tests, and add it to
  the cron next to `scorecard.py`.
- **L5, baseline universe.** Confirm which markets `scoring.baselines_forecast` covers. If it is not highs, lead 2,
  after 15:00, excluding HK and Taipei, open a scoring-fix CO.

## Rollback

```bash
cd /opt/data && patch -R -p1 < /path/to/CO-006.diff && cd /opt/data/polybot
.venv/bin/python tools/scorecard.py --register "CO-006 rolled back"
```

The 8 added columns stay in the table: SQLite keeps them, and CO-005's code no longer writes them, so new rows have
NULL there. No data is lost.

## CHANGELOG entry

```
## 2026-11-16 — CO-006: blend evaluation replaced; decision logging; freshness claim corrected (approved by Chase)
Audit AUDIT_CO-005_blend_2026-11-16. PREREGISTRATION section 7 replaces the section 6 stopping rule: S1 blend vs market
log-loss, S2 blend vs retired forecast pool, S3 per-share P/L of every >= 0.08 signal; futility at 30 effective dates,
final at 60. blend_snaps gains input_age_min, hours_to_peak, tradable, edge_yes, edge_no, fee_rate, model_name,
blend_version; input refresh failures are printed. tempmodel.model_lead docstring corrected (live inputs are older than
the trained lead, not fresher). No parameter, version or epoch change. Manifest re-registered.
```
