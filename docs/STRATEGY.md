# The strategy

## The market

Polymarket lists, for each city and day, "Highest temperature in <city> on <date>?". Each event is split into buckets
of 1 °C or 2 °F, one market per bucket. Exactly one bucket resolves YES, on the airport weather station's reports.

Costs that matter:

| Cost | Size |
|---|---|
| Taker fee | `shares × 0.05 × price × (1 − price)`, on every weather market since 2026-11-03 |
| Spread two days ahead | 5 to 8 cents |
| Minimum order | 5 shares |

## The current strategy: `blend`

Live on paper since 2026-11-14. Full rule: [CO-005](../change_orders/2026-11-14/CO-005-blend-strategy.md), as amended by
[CO-006](../change_orders/2026-11-16/CO-006-master-blend-execution-scoring-settlement.md). CO-006 was approved on 2026-11-16; its application was not yet confirmed when this was written, and items marked (CO-006) below depend on it.

```
 nine weather models ──▶ tree model ──▶ mu, sigma ──▶ probability per bucket ─┐
                                                                              ├─▶ log pool ──▶ blend probability
 order book ──▶ mid price per bucket, normalised over the event ──────────────┘         │
                                                                                        ▼
                                              buy a side when  blend − ask − fee ≥ 0.08
```

1. **Model.** Predict the station's daily high as a mean and a spread. See [MODEL.md](MODEL.md).
2. **Model probabilities.** A normal curve over each bucket's range, with the market's rounding rule.
3. **Market probabilities.** Mid price per bucket, normalised so the event sums to one.
4. **Blend.** A log pool with equal weight: `p ∝ model^0.5 × market^0.5`. The weight was not tuned.
5. **Trade.** Daily highs, two days ahead only, from 18:00 station time (CO-006; 15:00 before). Buy YES or NO at the ask when the blend's
   probability exceeds ask plus fee by at least 8 cents. Ask between 0.05 and 0.77 (CO-006; 0.90 before).
6. **Limits.** $2 per trade, $4 per event, $200 open in total. Fees count toward the limits, so in practice the bot
   takes one position per event, the one with the largest edge.
7. **Log everything.** Every bucket evaluated is recorded whether traded or not, so the result can be scored against
   the market with exact timing.

Why two days ahead only: one day ahead the blend scores worse than the market, so that lead is logged and scored but
never traded.

## Why a blend and not the model alone

The market's price is a better forecast than the model at every lead tested. The only measured advantage is the pool
of the two, two days ahead. See [RESULTS.md](RESULTS.md).

## Earlier strategies

| Strategy | Idea | Outcome |
|---|---|---|
| `intraday` | Same-day forecast from morning station readings | Retired 2026-10-25. The market beat it at every hour. 31 settled, 14 won, −$3.11 |
| `follow` | Late in the day, buy outcomes the thermometer has already settled, at 0.85 to 0.97 | Restricted to observation-only by CO-001 after forecast-dependent entries failed in three cities in one afternoon. Then nothing settled in two weeks. Retired 2026-11-14 |
| `forecast` | Ensemble forecast pooled with the market price | Last scoring period: 295 settled, 168 won against 162.7 implied by the prices paid, −$2.13. Its wins tracked the price, not its own forecast. Retired 2026-11-14 |

## What the research said before `blend` was built

| Finding | Report |
|---|---|
| The passive side of trades loses about 3.6% two days ahead; takers earn | [speed_lag.md](../reports/speed_lag.md) |
| The market moves before the weather centres publish the runs the old bot waited for, so speed was not the gap | [arrival_alignment.md](../reports/arrival_alignment.md) |
| Adding seven more weather models and forecast weather variables is what improved the temperature forecast | [temperature_baselines.md](../reports/temperature_baselines.md) |
