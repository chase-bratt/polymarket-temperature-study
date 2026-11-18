# CO-002: score with a date-clustered error band and a distinct-dates gate

**Status:** proposed 2026-10-25. **Risk:** none to trading. Measurement and display only; no strategy epoch changes.
**Patch:** `CO-002.diff` (unified diff, `patch -p1` from the bundle root). **Apply after CO-001** (its `config.py` hunk is cut against the CO-001 state; standalone it still applies with `patch -F2`).
**Files:** `polybot/polybot/governance.py`, `polybot/polybot/config.py`, `polybot/polybot/dashboard.py`, `hermes/plugins/polybot/dashboard/dist/index.js`.

## Why

The 95 % band `ROI ± 2·SE` treats every settled trade as an independent draw. They are not: all trades on one event date share one weather regime, and within a region they share the same forecast bust. At the snapshot forecast's current epoch is 30 trades on **2** event dates (24 on 2026-10-24); on that date Seoul and Busan both lost at z ≈ −1.5 and Beijing and Wuhan both won at z ≈ −2.3. "+24 % ± 32 %" reads as 30 observations; it is closer to two. The verdict rule (n ≥ 30 → judge) can declare "profitable" on a lucky week.

## What changes

1. `governance._band(rets, dates=None)`: SE becomes `max(sample SE, Laplace zero-loss floor, cluster-robust SE by event date)`. New `_cluster_se`: clusters = distinct event dates, `SE = sqrt(G/(G−1) · Σ_g (Σ_{i∈g}(r_i − r̄))²) / n`; `None` with fewer than 2 dates. Existing floors are kept, so the band can only get wider.
2. `scorecard` joins each position to its event date (`positions → markets → events.date`) and reports `dates` (distinct event dates) per epoch, plus `min_verdict_dates` at the strategy level.
3. New config `min_verdict_dates: int = 15` (not a `follow_*`/`FORECAST_KEYS` key, so it changes no epoch hash).
4. Dashboard: strategy card verdict reads "too few trades to judge (n/30 trades, d/15 event dates)" until **both** thresholds are met; epoch table gains a `dates` column. `python -m polybot governance` prints `dates` per epoch and the gate status.
5. Kill rule untouched: it still needs ≥ 100 settled and the whole band below zero; because the band is now wider, it fires later, never sooner. If Chase wants the dates gate on the kill rule too, add `and c["dates"] >= cfg.min_verdict_dates` in `enforce_kill` (one line; not included).

Sanity numbers on the snapshot's settled trades (computed with the patched functions): forecast current epoch 30 trades / 2 dates → band unchanged at ± 32 % (two clusters give a tiny clustered term; the dates gate does the work there), verdict becomes "too few trades to judge (30/30 trades, 2/15 event dates)". follow 15 / 3 dates → ± 13 %, unchanged, gated the same way. With 15+ dates the clustered term becomes the binding one whenever same-date results move together.

## Apply (Hermes agent, workdir /opt/data/polybot)

```bash
patch -p1 --dry-run < CO-002.diff && patch -p1 < CO-002.diff
.venv/bin/python -m polybot governance        # read-only; prints dates per epoch
.venv/bin/python -m polybot dashboard         # or whatever cycle.sh runs to publish the JSON
node polybot/tools/render_test.js             # dashboard renders; verdict text shows the dates gate
```

No cron pause needed (nothing changes a decision). No config.json edit needed (default 15); add `"min_verdict_dates": 15` there only if you want it visible.

## Verify

- `governance` output: every epoch line has `dates N`; `kill check` line ends with `verdict needs >= 30 settled and >= 15 distinct event dates (N so far)`.
- Dashboard JSON (`strategies.<name>.dates`, `.min_verdict_dates`, epochs[].dates) present; card verdict for forecast reads "too few trades to judge (30/30 trades, 2/15 event dates)" at the snapshot state.
- Band for any epoch with ≥ 2 dates is ≥ the pre-patch band (compare `governance` output before and after; keep the before output in the CHANGELOG entry).
- `enforce_kill` behaviour unchanged: run the fake-scorecard test from the 2026-10-21 governance entry (−20 % ± 10 % retires; −5 % ± 8 % does not).

## Rollback

`patch -R -p1 < CO-002.diff`. No data changes to undo.

## CHANGELOG entry

```
## 2026-10-27 — governance: date-clustered band + dates gate (CO-002, measurement only)
Evidence: forecast's current epoch (+24% ± 32%) is 30 trades on 2 event dates, 24 on one; same-date trades share one
regime (Oct 24: Seoul/Busan both lost z~-1.5, Beijing/Wuhan both won z~-2.3). The band assumed independence.
Changes: SE = max(sample, Laplace zero-loss floor, cluster-robust by event date); scorecard reports distinct event
dates; verdict needs >= 30 settled AND >= min_verdict_dates (15); dashboard shows both gates and a dates column.
Kill rule unchanged (wider band => fires later, never sooner). No epoch hash affected. Before/after `governance` output: ...
```
