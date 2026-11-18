# CO-001: follow strategy trades only what the thermometer has settled

**Status:** proposed 2026-10-25, for Chase's approval. **Risk:** reduces follow's trade count; adds no exposure.
**Patch:** `CO-001.diff` (unified diff, `patch -p1` from the bundle root; verified to apply cleanly to the 2026-10-25 handoff copy, follow.py sha1 6dff84df…).
**Files:** `polybot/polybot/follow.py`, `polybot/polybot/config.py`, `polybot/config.json`. Nothing else.

## Why

follow's thesis is "the station's own readings have already settled the outcome". As built, the same-day feasible range is bounded above (highs) by max(4 models' remaining-hours forecast) + 1.5 °C, so a bucket can be called **locked** (buy YES) or **dead above the observed max** (buy NO) on the strength of a same-day forecast, not an observation.

Evidence from the ledger at the snapshot:

- Of the 30 settled post-bid-filter follow trades, **1** was settled by observation alone (NO on a bucket below the observed max). The other 29 depended on the forecast cap holding (same-day YES, NO above the max, NO below the min).
- On 2026-10-25 the cap failed in three cities in one afternoon (all passed the bid ≥ 0.50 filter): Chengdu NO 25°C at 12:46 local, then the bot's own range moved and it bought YES 25°C at 14:52 on the same bucket (one side must lose); Kuala Lumpur YES 31°C + NO 32°C at 13:49 and Wuhan YES 26°C + NO 27°C at 14:52, where the observed max later passed the YES bucket. A daily max cannot come back down, so those YES legs cannot recover.
- Break-even at the average entry (0.943) is ~94 %. At 19:1 payoff, one miss costs ~19 wins. The intraday strategy was retired on 2026-10-25 because the same-day market beats our same-day forecast at every hour; forecast-dependent follow trades are the same bet with a worse payoff shape.

Settled results are not the argument (30/30, +$8.54): the structure is. Open positions are not scored.

## What changes

1. New parameter `follow_obs_only` (default `True`) in `config.py`. While the station day is underway the remaining-hours forecast is **not** used:
   - highs: feasible range = [round(observed max), +∞). Only buckets entirely below the observed max are **dead**; only the open-ended top bucket ("X or higher") can be **locked**, once the observed max ≥ X. Normal buckets lock only after the day is over (≥ 20 routine obs, as before).
   - lows: mirror image.
   - `future_extreme` is not called in obs-only mode, so the "no fresh hourly forecast" gate no longer blocks follow.
2. `bucket_state` accepts `None` for an open side of the range (stored as NULL in `follow_snaps.range_lo/range_hi`; the forecast column `fut_val` is NULL in obs-only mode and the scan line prints `future obs-only`).
3. **Opposite-side guard:** a follow order is rejected (`hold the opposite side of this bucket`) when the strategy already holds the other side of the same market. Unreachable after (1) in normal operation, cheap insurance.
4. `follow-backtest` applies the same rule so before/after can be compared on history.
5. `follow_version` → `2026-10-27-obs-only` in both `config.py` and `config.json` (decision code changed → new scoring epoch). `follow_margin_c` stays in config (used when `follow_obs_only` is false).

Not changed: price band 0.85–0.97, bid ≥ 0.50, stakes $5/$10/$150, Hong Kong exclusion, local-hour and METAR freshness gates. Nothing in the forecast strategy is touched (a guard there would bump `forecast_version` and reset its 30-trade epoch; not worth it now).

## Governance

follow is frozen until **2026-10-30T00:15Z** (its epoch started early on Oct 23 for the Polymarket rule change). This is a structural change to decision code, not a parameter tune, so it is Chase's call whether it (a) waits for the Oct 30 window and the weekly review, or (b) is applied now as with the intraday retirement. Either way the dashboard will show ⚠ EARLY CHANGE if applied before Oct 30; accept it in the CHANGELOG as below.

## Apply (Hermes agent, workdir /opt/data/polybot)

```bash
# 1. pause the hourly cron (hermes cron pause "polybot hourly paper cycle") and wait for any running cycle to finish
# 2. backup
cp data/polybot.sqlite data/polybot.pre-co001.sqlite
# 3. apply from the directory that contains polybot/ and hermes/ (the bundle layout); in production the paths are
#    /opt/data/polybot/polybot/follow.py, /opt/data/polybot/polybot/config.py, /opt/data/polybot/config.json
patch -p1 --dry-run < CO-001.diff && patch -p1 < CO-001.diff
# 4. one manual cycle; both version bumps land in the same scan so exactly ONE new follow epoch starts
.venv/bin/python -m polybot cycle
.venv/bin/python -m polybot governance
# 5. resume the cron
```

## Verify (all must hold before resuming the cron)

- `python -m polybot governance` shows a new follow epoch (sha differs from `1eb90fd31e`) starting at the cycle time, and no second epoch after it.
- In the cycle output, follow lines read `future obs-only -> range R..None` (highs) or `None..R` (lows) for lead-0 events; day-over events still show a closed range.
- `SELECT state, COUNT(*) FROM follow_snaps WHERE ts >= '<cycle ts>' AND lead_days = 0 GROUP BY state` shows `locked` only for buckets with `range_hi IS NULL AND lo <= range_lo` (highs: the "or higher" bucket) and `dead` only below the observed max. Query: `SELECT fs.*, m.bucket FROM follow_snaps fs JOIN markets m USING(market_id) WHERE fs.ts >= '<cycle ts>' AND fs.lead_days = 0 AND fs.state = 'locked'` → every bucket text ends in "or higher" / "or below".
- No fill in this cycle is a follow YES on a lead-0 normal bucket; a `[opposite held]` rejection, if any, appears in `orders.reason`.
- `python -m polybot follow-backtest` runs without error (fewer entries than before is expected).
- Dashboard renders (`node polybot/tools/render_test.js` or open the tab); `follow_snaps` NULL ranges do not break the positions MTM fallback.

## Rollback

`patch -R -p1 < CO-001.diff`, restore `follow_version` (the diff does), run one cycle. The epoch created by the change stays in the ledger as history; do not delete it.

## CHANGELOG entry (paste into notes/CHANGELOG.md)

```
## 2026-10-27 — follow: observation-only range (CO-001)
Evidence: 29 of 30 post-filter follow wins depended on max(4 models, remaining hours) + 1.5C holding. 2026-10-25 the cap
failed in Chengdu (bot held NO 25C then YES 25C), Kuala Lumpur (YES 31C + NO 32C) and Wuhan (YES 26C + NO 27C), all past
the bid filter; the observed max went past the YES bucket, which cannot reverse. Same-day forecast bets at 19:1 are the
bet intraday was retired for.
Changes: follow_obs_only=True (same-day range bounded by observations only: highs open above, lows open below; normal
buckets lock only once the day is over); opposite-side guard; follow-backtest mirrors the rule; follow_version bumped
-> new epoch (EARLY CHANGE accepted: structural fix, approved by Chase). Stakes, price band, bid filter unchanged.
```
