# Results

All figures are paper trading or replay on archived market data. Intervals are 95%, from a bootstrap over whole dates.

## 1. Forecast quality against the market

Daily highs, 23 to 25 market dates, timing restricted to what was knowable. Lower log-loss is better.
Source: [edge_after_costs.md](../reports/edge_after_costs.md).

| Days ahead | Market | Model alone | Blend | Blend minus market | Dates blend wins |
|---|---|---|---|---|---|
| 1 | 1.275 | 1.487 | 1.317 | +0.04 (+0.03 to +0.06) | 3 of 25 |
| 2 | 1.504 | 1.592 | 1.462 | −0.04 (−0.06 to −0.02) | 19 of 23 |

The market beats the model alone at both leads. The blend beats the market only two days ahead. On the Brier score the
blend's advantage is about zero, so the gain sits in the tails.

## 2. Does it pay? Replay with the bot's real limits and fees

Source: [gate_fees_order_r2.md](../reports/gate_fees_order_r2.md).

| Rule | Return per dollar | Interval |
|---|---|---|
| CO-005 as launched, fresh inputs | −0.3% | −7.6% to +7.9% |
| CO-005 as launched, stale inputs | −2.2% | −9.0% to +4.6% |
| CO-006 master | +2.4% | −5.6% to +10.9% |

CO-006 is worth about +$30 over 23 dates in replay. No interval excludes zero, and the rule was chosen on the same 23
dates it is scored on.

## 3. The rule that decides it

Written before any live result under it. Full text in
[CO-006](../change_orders/2026-11-16/CO-006-master-blend-execution-scoring-settlement.md).

| Check | When | Outcome |
|---|---|---|
| Futility | 30 market dates | Retire if the blend does not beat the market's log-loss, or if the whole return interval is below zero |
| Pass | 60 market dates | All four must hold: log-loss interval better than the market, Brier no worse, better than the retired strategy, return interval above zero |
| Otherwise | 60 market dates | Not passed |

## 4. How likely is each outcome

Simulated with the bot's own scoring functions. Source: [stopping_rule_power.md](../reports/stopping_rule_power.md).

| True edge per share | Chance of a pass |
|---|---|
| 0 cents | 1% |
| 2 cents | 8% |
| 4 cents | 35% |
| 6 cents | 76% |
| 8 cents | 96% |

A strategy with no forecasting edge is retired at the 30-date check about half the time. The observed replay edge is
between 0.2 and 2.5 cents per share, so the expected verdict is "not passed".

## 5. Earlier strategies

| Strategy | Settled | Won | Profit | End |
|---|---|---|---|---|
| `intraday` | 31 | 14 | −$3.11 | Retired 2026-10-25 |
| `forecast`, last period | 295 | 168, against 162.7 implied by prices paid | −$2.13 | Retired 2026-11-14 |
| `follow`, after CO-001 | 0 in two weeks | | | Retired 2026-11-14 |

## 6. Live result for `blend`

Pending. The evaluation needs 60 market dates and started in mid November 2026. This section will be filled in when the
rule reaches a verdict, whichever way it goes.

## What this does and does not show

| Claim | Supported |
|---|---|
| The model improves on raw weather forecasts | Yes |
| The market price is a strong forecast that the model alone does not beat | Yes |
| Model plus market carries a small information advantage two days ahead | Yes, in log-loss only |
| That advantage covers the fee and the spread | Not shown |
| No profitable strategy exists in these markets | Not shown. One strategy family was tested |
