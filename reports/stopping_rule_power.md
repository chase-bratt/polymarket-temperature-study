# Power of the PREREGISTRATION §7 stopping rule

Archive: 23 dates, 940 events, 201 signals. Observed means: S1a -0.0715 (22/23 dates below zero), S1b -0.0074 (16/23), S2 -0.0546 (16/23), S3 +0.0211 per share (date sd of the per-date mean 0.185).

1000 simulated 60-date histories per row; verdicts as percentages.

| true S3 edge per share | forecasting edge (S1a, S2) | retired at 30 | retired at 60 | not passed at 60 | PASS |
|---|---|---|---|---|---|
| +0c | none (S1a = 0) | 50% | 12% | 38% | 0% |
| +0c | as observed | 1% | 0% | 98% | 1% |
| +2c | as observed | 0% | 0% | 92% | 8% |
| +4c | as observed | 0% | 0% | 65% | 35% |
| +6c | as observed | 0% | 0% | 24% | 76% |
| +8c | as observed | 0% | 0% | 4% | 96% |

Share of histories reaching 60 dates in which each PASS condition holds on its own:

| true S3 edge per share | forecasting edge | S1a band < 0 | S1b mean <= 0 | S2 mean < 0 | S3 band > 0 |
|---|---|---|---|---|---|
| +0c | none (S1a = 0) | 5% | 100% | 0% | 1% |
| +0c | as observed | 100% | 100% | 100% | 1% |
| +2c | as observed | 100% | 100% | 100% | 8% |
| +4c | as observed | 100% | 100% | 100% | 35% |
| +6c | as observed | 100% | 100% | 100% | 76% |
| +8c | as observed | 100% | 100% | 100% | 96% |
