# CO-004: attribute METARs to station days by their observation time

**Status:** proposed 2026-10-30, for Chase's approval. The daily review asked for this on Oct 26, 27, 28 and 29 (its
P1). **Class:** breakage fix under PREREGISTRATION §1 exception 2 (the bot reads a wrong observed extreme on some days).
It changes follow's input, so `follow_version` is bumped → new follow epoch. **Cost of the bump: zero right now.**
follow has 0 settled and 0 open trades in its current epoch (2 bids posted since Oct 25, both expired).
**Patch:** `CO-004.diff` (`patch -p1` from the directory that holds `polybot/`; verified against the 2026-10-30 export).
**Files:** `polybot/polybot/analyze.py`, `polybot/polybot/follow.py`, `polybot/polybot/review.py`,
`polybot/polybot/config.py`, `polybot/config.json`. No locked file; the manifest is untouched.

## Why

`observations.report_time` is aviationweather's nominal hour: a report taken at 05:53Z is stored as 06:00Z. Every
reader bucketed reports into station-local days by that nominal time, so the hh:53 report taken just before local
midnight lands on the NEXT day. Evidence from the reviews (not re-derived here; the ledger is not in the export):
bucketing by the raw DDHHMMZ observation time matched the official result on 198/198 resolved METAR-rule events;
report_time matched 195/198 (Denver high and low 2026-10-24, Amsterdam low 2026-10-25). Denver low 10-24 is also the
likely cause of one of follow-backtest's two losses (a false "dead" at 00:00 local).

The live risk is in follow: on a frontal evening, yesterday's 23:53 reading can be the warmest of the "observed so far"
set for today, so a bucket below it is called dead and NO is bid on a bucket that can still win. Unit check in this
order (synthetic Denver day, hourly :53 reports, previous day's 23:53 report at 35 °C): old rule observed max 35.0,
new rule 24.0 with the same 24-report count.

## What changes

1. `analyze.obs_time(report_time, raw)`: parses `DDHHMMZ` from the start of the raw METAR (`[METAR|SPECI] ICAO DDHHMMZ`),
   resolves the day against report_time's month (handles month rollover), and falls back to report_time when the
   group is missing, malformed, or more than 3 h away from it. 9 unit cases pass (normal, SPECI, METAR prefix,
   exact hour, empty raw, far-off, malformed, month rollover).
2. `analyze.actuals`: day membership by `obs_time`. Everything that uses `actuals` follows automatically: `outcomes`
   (fit, analyze, follow-backtest), `shadow.load_errors`, `review`.
3. `follow.observed`: day membership and `last_obs` by `obs_time` (the freshness gate now measures from the real
   observation time, up to 7 min older than before). The query window is unchanged: nominal time is never earlier than
   the observation, so no report of the day can fall outside it.
4. `review._station_day_obs`: window extended by 2 h and filtered to the local day by `obs_time` (it previously took
   every report with a nominal time inside the UTC span of the day and did not filter by date).
5. `follow_version` `2026-10-25-obs-only-passive` → `2026-10-30-obs-time` in `config.py` and `config.json`.

Not changed, deliberately: `capture.py` and the `observations` primary key (nominal `report_time` stays the key; raw
is what carries the truth); `review.py`'s "typical daily report count" (still by nominal date; off by at most one
report, harmless); `intraday.py` (retired, snapshots only, not in the export; if its snapshots should use the same
rule, patch its day filter the same way in a separate step); `tools/backfill_history.py` (IEM archive has true times).

Effects on derived data: `actuals` changes for the three events above (and any future ones like them); `trade_reviews`
rows for Denver 10-24 and Amsterdam 10-25 keep their old `data` flag until re-reviewed (`python -m polybot review`
re-reviews only flagged rows without notes; if you want those two corrected, delete their trade_reviews rows and rerun:
trade_reviews is derived, not ledger). Shadow biases shift by a few hundredths of a degree for those stations. No
positions, fills, orders or cash rows change.

## Apply (Hermes agent, workdir /opt/data/polybot)

```bash
# 1. pause the hourly cron and wait for any running cycle to finish (follow.py is edited; cycle.sh is not)
# 2. backup
cp data/polybot.sqlite data/polybot.pre-co004.sqlite
# 3. apply from the directory that contains polybot/ (production: /opt/data)
cd /opt/data && patch -p1 --dry-run < CO-004.diff && patch -p1 < CO-004.diff && cd /opt/data/polybot
.venv/bin/python -m py_compile polybot/analyze.py polybot/follow.py polybot/review.py polybot/config.py
# 4. one manual cycle: exactly ONE new follow epoch starts (config.py default and config.json change together)
./cycle.sh
.venv/bin/python -m polybot governance
# 5. resume the cron
```

## Verify

- `governance`: follow shows a new epoch from the cycle time (sha differs from 1e8913f6b5), no second epoch after it,
  and the old epoch keeps its 0 settled. EARLY CHANGE is expected (follow's last change was Oct 25); accept it in the
  CHANGELOG as a breakage fix.
- `python -m polybot analyze` prints the METAR-inferred vs resolved agreement: it should read 198/198 (or the current
  total/total) instead of 195/198.
- `python -m polybot follow-backtest` runs; the Denver low 10-24 loss should disappear (one fewer loss, entries about
  the same).
- Spot check in sqlite: for a US station, `SELECT report_time, raw FROM observations WHERE icao='KDEN' ORDER BY
  report_time DESC LIMIT 3` shows raw `DDHHMMZ` seven minutes before `report_time`; the next cycle's follow line for
  that station shows obs n equal to the number of routine reports taken that local day.
- Dashboard renders; follow_snaps rows for the cycle have `obs_n` unchanged or one lower than the previous cycle at
  the same local hour (the previous day's 23:53 report is no longer counted).

## Rollback

`patch -R -p1 < CO-004.diff`, run one cycle (restores `follow_version`; the epoch created stays in the ledger as
history, do not delete it).

## CHANGELOG entry (paste into notes/CHANGELOG.md)

```
## 2026-10-30 — observations attributed to days by DDHHMMZ obs time (CO-004, breakage fix; follow epoch bumped)
Evidence: observations.report_time is aviationweather's nominal hour (0553Z stored as 06:00Z), so the hh:53 report
before local midnight was counted on the next station day. By obs time the inferred extreme matched the official result
198/198 METAR-rule events; by report_time 195/198 (Denver hi+lo 10-24, Amsterdam lo 10-25). A frontal-day 23:53 reading
could inflate follow's "observed max so far" and call a live bucket dead.
Changes: analyze.obs_time (raw DDHHMMZ, fallback report_time); analyze.actuals, follow.observed, review._station_day_obs
bucket by obs time; shadow, fit, analyze, follow-backtest inherit it. follow_version -> 2026-10-30-obs-time (new epoch;
EARLY CHANGE accepted: breakage fix approved by Chase; the previous epoch had 0 trades). Capture, PK and ledger untouched.
```
