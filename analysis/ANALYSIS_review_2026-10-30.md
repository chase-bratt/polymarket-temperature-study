# polybot review of the 2026-10-30 export

Source: `polybot_export_2026-10-30/` (since_2026-10-25_0920, performance_all_time, strategy code and notes) and
`ANALYSIS_forecast_2026-10-30.md`. Every number below was recomputed from the CSVs or read from the code in the export.
Only items I am highly confident in are listed as changes or errors.

## 1. The Oct 30 forecast analysis checks out

Reproduced exactly from `settled_trades.csv` and `resting_bids.csv`: d5e06839b5 137 / 83 won / price 79.4 / pool
95.2 / +$16.90; f312e9d5ff 53 / 28 / 29.8 / 37.7 / −$13.67; by date 1/1, 12/30 (−$19.76), 15/22; 2.06 effective dates;
filled bids 53 / 28 won / price 30.5; unfilled 70 / 55 / 43.7 / +26% counterfactual; 55 open ($153.49).
One small difference: filled bids at the bid price come to −8% per share (booked −10%), not −14%. Conclusions unchanged.

## 2. Forecast: the pool is biased, the market is not (high confidence)

Three independent measurements agree:

| measurement | favourite won | NO price implied | pool implied |
|---|---|---|---|
| trade-free, live rule, 238 cases, 8 dates | 99 | 103.4 | 75.0 |
| old epoch, 137 settled | 83 of our sides | 79.4 | 95.2 |
| new epoch, 53 settled | 28 of our sides | 29.8 | 37.7 |

The pool overstates the probability of the side it buys by about 10 points every time; the price is within noise
(238 cases: market off by 0.6 SE, pool off by 3.2 SE). The 0.08 + drift bar is cleared by that bias, not by an edge.
The unconditional calibration table (803 favourites, 11 dates) shows the same.

**Change I recommend (risk-reducing, allowed in the freeze): pause forecast entries, keep snapshots and shadow.**
Nothing is lost by pausing: the conditional trade-free test runs on snapshots without any orders and is the test the
pool thesis lives or dies on. Trading only adds execution data, and that answer is already in (unfilled bids win 79%,
filled 53%: adverse selection). Kill rule needs 100 settled; at ~25 settled per trading day that is 3 more weeks of
paper losses to learn what the snapshots already say. Your call under the pre-registration.

**Research before any model change (allowed now, trade-free):** `strategy.evaluate_event` builds p_mkt from the ask
alone on no-bid buckets (`mids.append(... ask if ask is not None else 0.0005)`), then normalises. With ~73% overround on
asks this inflates the tails and deflates the favourite, which is exactly the direction of the bias above. Recompute
p_mkt on the stored snapshots with no-bid buckets at ask/2 (or the floor), re-pool, and re-run the conditional
calibration. If the 75-vs-99 gap closes, the fix is the de-vig, not the weather model.

## 3. Follow is dormant and cannot produce a verdict (high confidence, fact)

Since the passive epoch began (Oct 25 16:23Z): 2 bids posted (Buenos Aires NO, Oct 25 and Oct 29), both expired, 0 fills,
0 settled. Scorecard: B3 found 549 market-sides in follow's price universe in the same window; B4 (follow's own state
check on the same snapshots) found 4. Observation-only + ask 0.85–0.97 + own bid ≥ 0.50 almost never coincide: by the
time the thermometer has settled a bucket the market is at 0.98+. Also from the scorecard, since the Oct 21 bid filter:
B3 (85–97¢ market-backed side, no thermometer) 1037 trades, −3.8%, band −6.6% to −1.1% (8.1 eff. dates); B4 (follow rule
replayed) 48 trades, 43 won, −4.9%. The universe loses passively and the thermometer has not been shown to beat it.
No loosening is allowed in the freeze and none is justified by the data. Decision for you: leave follow dormant or
retire it. Either way, do not count on follow for the Dec 20 verdict.

## 4. Errors in the locked scoring code (re-registration needed to fix)

1. **Capacity criterion can never pass under the passive broker.** `scoring.fill_realism` measures capacity as BUY prints
   of our token at ≤ our price within −10/+30 min of the fill timestamp. Maker fills are stamped at the cycle that
   matched them and were caused by SELL-side (or complement) prints up to 30 min earlier, so print_size is 0 for every
   maker fill: scorecard shows "capacity median 0.0x" on 108 confirmed fills. Fix: for `maker=1` fills, capacity =
   qualifying tape volume over the resting window (`execution._qualifying(posted, expiry)`) / shares.
2. **`scoring.t_crit` is inaccurate where it matters.** Hill expansion vs exact two-sided 95% t: df 1.5 → 4.80 vs 6.02
   (−20%), df 2 → 3.85 vs 4.30 (−10%), df 3 → 3.06 vs 3.18 (−4%); df 1.06 → 12.71 vs 11.13 (+14%). The band is too
   narrow exactly in the 2.5–4 effective-date range the strategies are entering. Fix: exact inverse t (scipy or a table).
3. **Two different "95% bands" for the same epoch.** Dashboard/SUMMARY and the kill rule use roi ± 2·SE (forecast
   −32.9% to +12.9%); the registered scorecard uses t(df = eff dates − 1) × SE (−273.7% to +252.2%). Both are shown to
   you for the same 53 trades. Not a bug in either, but label them differently, and note that the registered band
   cannot fire the kill rule until ~10 effective dates while the 2·SE band can.
4. **B2 baseline pays the NO ask; the strategy fills at or below mid.** "Beats its baseline" is therefore easier than
   intended (B2 −19.2% is mostly spread: selling the favourite at the NO mid loses only ~2%). Score B2 at the NO mid.

Items 1, 2 and 4 are measurement only; none changes a decision. The cluster-SE term never binding (7/7 epochs) is
already acknowledged in the CHANGELOG; the effective-dates gate does that work.

## 5. Pending breakage fix that should be approved now

The daily review has asked three times (Oct 26–29) to attribute METARs to days by the raw DDHHMMZ observation time:
198/198 official results matched vs 195/198 by report_time (the hh:53 report before local midnight lands on the wrong
day). It changes follow's observed-max input, so it bumps follow's epoch. Follow has zero trades in its epoch, so the
cost of that bump is zero right now; it will not be later.

## 6. Verified clean

Ledger arithmetic on 176 new settled rows (pnl = payout − cost − fee, fee 0 on all maker fills); every posted forecast bid
has edge ≥ 0.08 + drift (723 bids, min 0.101) and edge = p_model − limit; drift 0.019–0.042; no both-sides holdings within
a strategy; no cross-strategy overlap in open positions; `execution.quote` / `match` logic as documented.
