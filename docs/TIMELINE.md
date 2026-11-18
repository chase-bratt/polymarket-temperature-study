# Timeline: every change from first to current

```
Oct 19        Oct 25           Oct 30          Nov 13         Nov 14        Nov 16
  │             │                │               │              │             │
 bot starts   intraday retired  scoring and    data pipeline  blend goes    three audits,
 3 strategies CO-001, CO-002    station fixes  model trained  live (CO-005) master CO-006
                                CO-003, CO-004
```

## Change orders

A change order is a written case for one change: the evidence, exactly what changes, how to apply it, how to verify
it, and how to undo it. The owner approves or rejects each one.

| Order | Date | What changed | Why | Status |
|---|---|---|---|---|
| [CO-001](../change_orders/2026-10-25/CO-001-follow-observation-only.md) | 2026-10-25 | `follow` decides from station observations only, never from a same-day forecast | 29 of 30 wins had depended on a forecast cap that failed in three cities in one afternoon | Applied |
| [CO-002](../change_orders/2026-10-25/CO-002-date-clustered-band.md) | 2026-10-25 | Error bands cluster by date; a verdict needs at least 15 distinct dates | 30 trades on 2 dates were being read as 30 independent results | Applied |
| [CO-003](../change_orders/2026-10-30/CO-003-scoring-fixes.md) | 2026-10-30 | Exact Student-t in the band; fill capacity measured correctly; fairer baseline | Three defects in the official scorecard | Applied |
| [CO-003-ALT](../change_orders/2026-10-30/CO-003-ALT-comprehensive-math-and-execution-fixes.md) | 2026-10-30 | A separate, broader proposal filed the same day | Alternative to CO-003 | Not adopted as written |
| [CO-004](../change_orders/2026-10-30/CO-004-metar-obs-time.md) | 2026-10-30 | Station reports assigned to days by their true observation time | A late-evening report could be counted in the wrong day | Applied |
| [CO-005](../change_orders/2026-11-14/CO-005-blend-strategy.md) | 2026-11-14 | New strategy `blend` with the trained model; `forecast` and `follow` retired | The old strategy had no edge; a better model existed | Applied 2026-11-14 |
| [CO-006](../change_orders/2026-11-16/CO-006-master-blend-execution-scoring-settlement.md) | 2026-11-16 | Fees in the edge and the limits, 18:00 start, best-edge-first ordering, safer settlement, a pass rule with real power | Findings of three audits, merged | Approved 2026-11-16; application not yet confirmed when this was written |

## What happened, in order

**2026-10-19.** The bot starts paper trading with three strategies: `forecast`, `follow`, `intraday`.

**2026-10-25.** `intraday` is retired: the same-day market beat the same-day model at every hour. The first review
([analysis](../analysis/ANALYSIS_2026-10-25.md)) finds that `follow` was betting on forecasts while claiming to bet on
observations, and that the scorecard overstated its certainty. CO-001 and CO-002 fix both.

**2026-10-30.** Second export reviewed ([analysis](../analysis/ANALYSIS_review_2026-10-30.md)). The scorecard has three
measurement faults and the station-day logic one. CO-003 and CO-004 fix them. No trading rule is loosened.

**2026-11-13.** Third export reviewed ([analysis](../analysis/ANALYSIS_review_2026-11-13.md)). `forecast` stands at 295
settled bets and −$2.13, with wins tracking the prices paid. A data pipeline is built and audited
([data audit](../reports/DATA_AUDIT_2026-11-13.md)). Two research questions are answered: speed is not the gap, breadth
of weather models is.

**2026-11-13 to 14.** The temperature model is trained, tuned on validation dates, and tested once. Against market
prices the blend has a small forecasting advantage two days ahead and no proven trading edge. CO-005 states this
plainly and launches `blend` as a paper test with a stopping rule written in advance.

**2026-11-16.** Early `blend` results look like the retired strategy's. Three independent audits are commissioned and
cross-examined ([AUDITS.md](AUDITS.md)). They find real faults: fees left out of the replay, a start time that did not
match what was tested, a stopping rule too weak to detect a working strategy, and a settlement path that could pay on
a non-final result. The master CO-006 merges the confirmed findings and discards the unsupported ones.

**Now.** The 60-date evaluation is running. The result will be recorded here whichever way it goes.
