# Settlement and maker-fill checks on the 2026-11-13 ledger

## A4a. YES outcomes per fully resolved event (must be exactly 1)

```
 yes_outcomes  events
            1    2224
```

## A4b. Partly resolved events (some buckets resolved, some not) and their YES counts

```
 partly_resolved  partly_with_two_yes  unresolved  fully_resolved
              17                    0         278            2224
```

## A4c. Bot outcome vs the bucket the METAR observations imply (complete observations only)

```
 proxy_station  resolved_events  agree  disagree  agree_pct
         False             2127   2122         5      99.76
          True               96     60        36      62.50
```

## A4d. Markets marked resolved before the station day ended (premature closure)

```
 resolved_markets  before_day_end  min_hours_after_day_end  median_hours_after
            24493               0                      0.3                 1.0
```

## A6a. Tape by taker direction

```
taker_side  prints     shares  pct
      SELL  221591  6774178.0 14.0
       BUY 1364597 37726203.0 86.0
```

## A6b. The bot's maker fills: tape support under the current rule and under a direction-aware rule

```
 filled_resting_orders  supported_current_rule  supported_direction_aware  lost_if_direction_aware  supported_by_trade_through_alone  shares_filled  shares_still_filled_direction_aware
                   447                     447                        396                       51                               409         2332.0                               2056.0
```

## A3. Fee on top of the stake in archived taker fills (the amount the cap checks ignore)

```
strategy  taker_fills  max_fee_usd  avg_fee_usd  max_fee_pct_of_stake  total_fee_usd
forecast          208       0.0994       0.0681                  4.97          14.16
```

## A3b. Lowest cash balance ever recorded

```
 min_cash_balance
            50.61
```
