# Deferred model ideas: is any of them a measurable improvement?

Daily highs. Rule set before running: ship only what improves the test-split score with a 95% interval excluding
zero and does not worsen the blend's market scores. Intervals: bootstrap over whole dates; `*` excludes zero.

## 1. Proxy stations in or out of training (scored on market dates, non-proxy stations)

| lead | training set | rows | RMSE °C | bias | in 80% band | in 95% band | mean z² | beyond 2.5 sd | NLL |
|---|---|---|---|---|---|---|---|---|---|
| 1 | without proxies (research) | 1,296 | 0.990 | +0.070 | 83.5% | 95.9% | 0.87 | 0.77% | 1.3562 |
| 1 | with proxies (exported package) | 1,296 | 0.992 | +0.077 | 83.1% | 96.6% | 0.86 | 0.77% | 1.3557 |
| 1 | NLL, with minus without | 1,296 | | | | | | | -0.00 (-0.01 to +0.01) |
| 2 | without proxies (research) | 1,296 | 1.121 | +0.095 | 83.3% | 96.5% | 0.89 | 1.16% | 1.4909 |
| 2 | with proxies (exported package) | 1,296 | 1.119 | +0.104 | 82.8% | 96.0% | 0.89 | 1.00% | 1.4876 |
| 2 | NLL, with minus without | 1,296 | | | | | | | -0.00 (-0.01 to +0.00) |

## 2 and 3. Tail shape and spread, out of sample

Degrees of freedom chosen on the validation split (model fitted on train only):

| lead | t(3) | t(4) | t(5) | t(6) | t(8) | t(10) | t(15) | t(20) | t(30) | normal | chosen |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 1.5071 | 1.4691 | 1.4603 | 1.4575 | 1.4567 | 1.4575 | 1.4601 | 1.4624 | 1.4654 | 1.4785 | 8 |
| 2 | 1.6819 | 1.6279 | 1.6127 | 1.6066 | 1.6027 | 1.6021 | 1.6038 | 1.6060 | 1.6094 | 1.6229 | 10 |

| lead | scored on | rows | RMSE °C | bias | in 80% band | in 95% band | mean z² | beyond 2.5 sd | NLL normal | NLL t minus normal (95%) |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | test split (model fitted on train + val) | 10,368 | 1.118 | -0.009 | 85.1% | 96.4% | 0.85 | 1.43% | 1.4581 | -0.02 (-0.03 to -0.01) * |
| 1 | market dates (fitted to 2026-10-09) | 1,296 | 0.990 | +0.070 | 83.5% | 95.9% | 0.87 | 0.77% | 1.3562 | -0.00 (-0.01 to +0.00) |
| 2 | test split (model fitted on train + val) | 10,368 | 1.287 | -0.030 | 85.1% | 96.2% | 0.86 | 1.27% | 1.5943 | -0.02 (-0.02 to -0.01) * |
| 2 | market dates (fitted to 2026-10-09) | 1,296 | 1.121 | +0.095 | 83.3% | 96.5% | 0.89 | 1.16% | 1.4909 | -0.01 (-0.02 to +0.00) |

A normal curve puts 80.0% inside the 80% band, 95.0% inside the 95% band and 1.24% beyond 2.5 sd; mean z² should be 1.00.

## Effect on the blend's market scores (lead 2, each event's first scan from 18:00, as the scorecard scores it)

| score | normal | t(10) | t minus normal (95%) |
|---|---|---|---|
| model alone, log-loss on the winning bucket | 1.4863 | 1.4845 | -0.00 (-0.01 to +0.00) |
| S1a blend minus market log-loss | -0.0779 | -0.0731 | +0.00 (+0.00 to +0.01) * |
| S1b blend minus market Brier | -0.0090 | -0.0094 | -0.00 (-0.00 to +0.00) |
| S3 signals | 207 | 229 | |
| S3 net return per share | +0.0248 | +0.0168 | |
| S3 95% interval | +0.02 (-0.03 to +0.08) | +0.02 (-0.05 to +0.08) | |
