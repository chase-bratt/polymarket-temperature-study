# The temperature model

A gradient-boosted decision tree model that corrects the error of a nine-model weather forecast, with a second set of
trees that predicts how uncertain the forecast is.

```
 mu    = mean of nine weather models + trees(inputs)        the trees predict the forecast's error
 sigma = sqrt(scale × exp(trees_var(inputs)))               a second forest predicts the size of the miss
```

Explore the trained trees in [model/tree_viewer.html](../model/tree_viewer.html) (download and open in a browser).
Settings: [model/model_params.json](../model/model_params.json).

## Data

| Item | Detail |
|---|---|
| Target | Daily high and low at 50 airport stations, from station reports |
| Forecasts | Archived runs of nine models: ECMWF, GFS, ICON, GEM, JMA, Météo-France, UKMO, CMA, BOM, from 2024-02 |
| Observations | Station history from 2000 |
| Models trained | Six: high and low, each for 1, 2 and 3 days ahead |

## Inputs (49)

| Group | Examples |
|---|---|
| Forecast | ECMWF and GFS values, multi-model spread, each other model's gap from the mean, forecast dewpoint, cloud, wind, pressure |
| Place and season | Latitude, longitude, day of year, station |
| Recent errors | The forecast's mean error over 7 and 30 days, yesterday's error, forecast revisions |
| Yesterday's weather | Analysed cloud, dewpoint, wind, pressure, rain and sunshine |

## Splits

Split by whole dates in time order, so stations on the same day never sit on both sides.

| Split | Dates | Use |
|---|---|---|
| Train | 2024-02-04 to 2025-06-30 | Fit |
| Validation | 2025-07-08 to 2026-02-28 | Choose inputs and settings |
| Test | 2026-03-08 to 2026-10-09 | Opened once, after all choices were fixed |
| Market dates | after 2026-10-09 | Compare against market prices |

## Tuning (validation only)

Source: [temperature_tuning.md](../reports/temperature_tuning.md).

| Choice | Options tried | Chosen |
|---|---|---|
| Inputs | 32, 39 or 49 | 49 |
| Trees | Nine settings | 300 trees, learning rate 0.05, 15 leaves, at least 100 rows per leaf |
| Spread | Formula from model disagreement, or a second forest | Second forest: 150 trees, 8 leaves |

The second forest for the spread was the largest single gain: bucket log-loss 1.571 to 1.537.

## Baseline ladder

Daily high, one day ahead, validation window. Source: [temperature_baselines.md](../reports/temperature_baselines.md).

| Method | Error (RMSE, °C) | Bucket log-loss |
|---|---|---|
| Climatology | 3.86 | 2.681 |
| Yesterday's value | 4.05 | 2.812 |
| Raw ECMWF | 1.80 | 1.998 |
| Raw mean of models | 1.52 | 1.847 |
| Station bias correction | 1.31 | 1.657 |
| 30-day running bias | 1.23 | 1.630 |
| Trees, first version | 1.18 | 1.555 |
| **Final model** | **1.12** | **1.474** |

## Test window result

| Lead | Raw nine-model mean | Tuned model |
|---|---|---|
| 1 day ahead | 1.52 °C | 1.13 °C |
| 2 days ahead | 1.65 °C | 1.30 °C |

## Ideas tested and not adopted

Rule fixed before running: adopt only what improves the test score with a 95% interval excluding zero and does not
worsen the scores against the market. Source: [model_variants.md](../reports/model_variants.md).

| Idea | Result | Decision |
|---|---|---|
| Train without two stations whose labels come from a proxy source | No difference: error 1.121 against 1.119 °C | Not adopted |
| Fatter-tailed curve (Student-t) in place of the normal | Better on the test window, worse against the market | Not adopted |
| Widen or narrow the predicted spread | Spread is already slightly wide: 83% to 85% of results inside the 80% band | Left alone |

## Limits found by audit

| Limit | Size |
|---|---|
| Live inputs are about 6 hours staler than the training archive early in the trading window | A full 24 hours of staleness raises error from 1.12 to 1.28 °C; the live effect is roughly a quarter of that |
| A better forecast score than the market is not proof of profit | The retired strategy had a similar score advantage two days ahead and made nothing |
