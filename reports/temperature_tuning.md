# Temperature model tuning (validation dates only)

Train 2024-02-04 00:00:00 to 2025-06-30 00:00:00; validation 2025-07-08 00:00:00 to 2026-02-28 00:00:00. Bucket log-loss on whole-degree readings, lower is better. The test window was not read.

## A. Which inputs

| setting | tmax L1 RMSE | log-loss | 80% cover | tmax L2 RMSE | log-loss | 80% cover | mean log-loss |
|---|---|---|---|---|---|---|---|
| forecast (32 inputs) | 1.148 | 1.531 | 82% | 1.322 | 1.654 | 82% | **1.5926** |
| plus_recent (39 inputs) | 1.132 | 1.518 | 82% | 1.307 | 1.643 | 82% | **1.5801** |
| full (49 inputs) | 1.128 | 1.512 | 82% | 1.308 | 1.642 | 82% | **1.5767** |

## B. Forest settings

| setting | tmax L1 RMSE | log-loss | 80% cover | tmax L2 RMSE | log-loss | 80% cover | mean log-loss |
|---|---|---|---|---|---|---|---|
| defaults | 1.128 | 1.512 | 82% | 1.308 | 1.642 | 82% | **1.5767** |
| slower, 600 trees | 1.132 | 1.515 | 82% | 1.312 | 1.646 | 82% | **1.5806** |
| 600 trees | 1.128 | 1.512 | 82% | 1.313 | 1.646 | 82% | **1.5788** |
| bigger trees (31 leaves) | 1.130 | 1.514 | 82% | 1.310 | 1.646 | 82% | **1.5799** |
| smaller trees (7 leaves), 600 | 1.125 | 1.509 | 82% | 1.308 | 1.642 | 82% | **1.5757** |
| leaves hold >= 100 rows | 1.121 | 1.504 | 83% | 1.303 | 1.639 | 82% | **1.5714** |
| stronger shrinkage (l2 = 10) | 1.127 | 1.510 | 82% | 1.304 | 1.639 | 82% | **1.5748** |
| half the inputs per split | 1.127 | 1.511 | 82% | 1.309 | 1.644 | 82% | **1.5775** |
| half the inputs, slower, 600 | 1.121 | 1.506 | 83% | 1.309 | 1.643 | 82% | **1.5748** |

## C. Sigma

| setting | tmax L1 RMSE | log-loss | 80% cover | tmax L2 RMSE | log-loss | 80% cover | mean log-loss |
|---|---|---|---|---|---|---|---|
| spread | 1.121 | 1.504 | 83% | 1.303 | 1.639 | 82% | **1.5714** |
| forest | 1.121 | 1.470 | 83% | 1.303 | 1.603 | 82% | **1.5369** |

**Chosen:** inputs `full`, forest `leaves hold >= 100 rows` {"max_iter": 300, "learning_rate": 0.05, "max_leaf_nodes": 15, "min_samples_leaf": 100, "l2_regularization": 1.0}, sigma `forest`.
