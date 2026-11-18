# Temperature baseline ladder

Source: `data/ml/temperature/temp_hist_fc.parquet` (archived model forecasts vs station METAR extremes; rows where ECMWF and GFS both exist), 50 stations. Fitted on train only.

## val: 2025-07-08 00:00:00 to 2026-02-28 00:00:00 (train: 2024-02-04 00:00:00 to 2025-06-30 00:00:00)

### tmax, lead 1 (11,608 station-days, 236 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 3.86 | -0.26 | 2.062 | 2.681 | 13.2% | 81% |
| persistence | 4.05 | -0.08 | 2.198 | 2.812 | 14.8% | 85% |
| raw_ecmwf | 1.80 | +0.57 | 0.988 | 1.998 | 24.0% | 84% |
| raw_gfs | 1.84 | +0.31 | 1.011 | 2.033 | 25.5% | 85% |
| raw_mean | 1.52 | +0.71 | 0.851 | 1.847 | 27.3% | 78% |
| station_bias | 1.31 | -0.06 | 0.711 | 1.657 | 34.3% | 83% |
| running_bias | 1.23 | +0.02 | 0.672 | 1.630 | 37.3% | 84% |
| emos | 1.35 | -0.01 | 0.733 | 1.687 | 33.2% | 82% |
| trees | 1.18 | -0.06 | 0.636 | 1.555 | 38.8% | 83% |
| model | 1.12 | -0.03 | 0.599 | **1.474** | 39.9% | 83% |

### tmax, lead 2 (11,608 station-days, 236 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 3.86 | -0.26 | 2.062 | 2.682 | 13.2% | 81% |
| persistence | 4.45 | -0.13 | 2.428 | 2.910 | 12.8% | 82% |
| raw_ecmwf | 1.97 | +0.58 | 1.077 | 2.087 | 22.2% | 84% |
| raw_gfs | 2.02 | +0.15 | 1.117 | 2.131 | 22.3% | 85% |
| raw_mean | 1.68 | +0.73 | 0.934 | 1.941 | 25.6% | 79% |
| station_bias | 1.48 | -0.06 | 0.800 | 1.775 | 30.8% | 82% |
| running_bias | 1.40 | +0.03 | 0.764 | 1.758 | 33.1% | 85% |
| emos | 1.53 | -0.02 | 0.821 | 1.797 | 30.7% | 82% |
| trees | 1.35 | -0.08 | 0.725 | 1.678 | 34.6% | 83% |
| model | 1.30 | +0.08 | 0.693 | **1.607** | 35.4% | 82% |

### tmin, lead 1 (11,608 station-days, 236 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 3.44 | -0.30 | 1.813 | 2.541 | 15.1% | 77% |
| persistence | 3.49 | -0.09 | 1.869 | 2.655 | 17.9% | 83% |
| raw_ecmwf | 1.82 | +0.23 | 0.998 | 2.014 | 24.8% | 82% |
| raw_gfs | 2.49 | -0.25 | 1.339 | 2.321 | 20.6% | 82% |
| raw_mean | 1.55 | -0.04 | 0.852 | 1.857 | 28.5% | 82% |
| station_bias | 1.34 | -0.09 | 0.724 | 1.657 | 32.5% | 80% |
| running_bias | 1.25 | +0.01 | 0.686 | 1.641 | 35.6% | 83% |
| emos | 1.42 | -0.00 | 0.773 | 1.728 | 30.5% | 82% |
| trees | 1.20 | -0.07 | 0.650 | 1.577 | 37.3% | 84% |
| model | 1.15 | -0.02 | 0.618 | **1.498** | 38.6% | 82% |

### tmin, lead 2 (11,608 station-days, 236 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 3.44 | -0.30 | 1.813 | 2.541 | 15.1% | 77% |
| persistence | 3.87 | -0.13 | 2.074 | 2.758 | 15.2% | 81% |
| raw_ecmwf | 1.93 | +0.23 | 1.054 | 2.068 | 23.2% | 81% |
| raw_gfs | 2.62 | -0.88 | 1.414 | 2.382 | 19.7% | 79% |
| raw_mean | 1.64 | -0.13 | 0.897 | 1.911 | 27.2% | 82% |
| station_bias | 1.44 | -0.12 | 0.781 | 1.735 | 30.8% | 80% |
| running_bias | 1.35 | +0.01 | 0.744 | 1.721 | 32.5% | 83% |
| emos | 1.53 | +0.01 | 0.834 | 1.809 | 28.4% | 82% |
| trees | 1.31 | -0.10 | 0.716 | 1.675 | 33.8% | 83% |
| model | 1.27 | -0.02 | 0.680 | **1.587** | 35.3% | 82% |

## test: 2026-03-08 00:00:00 to 2026-10-09 00:00:00 (train: 2024-02-04 00:00:00 to 2025-06-30 00:00:00)

### tmax, lead 1 (10,762 station-days, 216 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 3.77 | +0.25 | 2.035 | 2.674 | 13.1% | 81% |
| persistence | 3.98 | +0.06 | 2.157 | 2.797 | 15.4% | 85% |
| raw_ecmwf | 1.92 | +0.47 | 1.050 | 2.058 | 22.6% | 81% |
| raw_gfs | 2.04 | +0.35 | 1.106 | 2.124 | 23.3% | 83% |
| raw_mean | 1.52 | +0.66 | 0.843 | 1.838 | 27.0% | 79% |
| station_bias | 1.27 | -0.12 | 0.691 | 1.636 | 35.2% | 84% |
| running_bias | 1.21 | -0.02 | 0.665 | 1.618 | 37.4% | 85% |
| emos | 1.35 | -0.05 | 0.734 | 1.689 | 33.2% | 84% |
| trees | 1.16 | -0.06 | 0.630 | 1.551 | 39.1% | 85% |
| model | 1.13 | -0.09 | 0.604 | **1.484** | 40.5% | 85% |

### tmax, lead 2 (10,762 station-days, 216 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 3.77 | +0.25 | 2.034 | 2.674 | 13.1% | 81% |
| persistence | 4.36 | +0.09 | 2.374 | 2.891 | 13.4% | 83% |
| raw_ecmwf | 2.07 | +0.48 | 1.134 | 2.135 | 21.0% | 81% |
| raw_gfs | 2.23 | +0.19 | 1.218 | 2.217 | 21.3% | 82% |
| raw_mean | 1.65 | +0.64 | 0.918 | 1.920 | 25.5% | 80% |
| station_bias | 1.43 | -0.15 | 0.777 | 1.746 | 31.6% | 83% |
| running_bias | 1.38 | -0.02 | 0.755 | 1.744 | 33.7% | 85% |
| emos | 1.51 | -0.02 | 0.818 | 1.793 | 30.4% | 84% |
| trees | 1.33 | -0.07 | 0.720 | 1.678 | 34.8% | 85% |
| model | 1.30 | -0.01 | 0.692 | **1.608** | 36.2% | 85% |

### tmin, lead 1 (10,762 station-days, 216 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 2.83 | +0.14 | 1.547 | 2.410 | 15.4% | 82% |
| persistence | 3.01 | +0.06 | 1.636 | 2.524 | 18.6% | 88% |
| raw_ecmwf | 1.62 | +0.19 | 0.898 | 1.906 | 26.9% | 85% |
| raw_gfs | 2.08 | +0.06 | 1.152 | 2.164 | 21.0% | 87% |
| raw_mean | 1.39 | -0.03 | 0.766 | 1.754 | 31.3% | 86% |
| station_bias | 1.22 | -0.09 | 0.662 | 1.575 | 34.8% | 83% |
| running_bias | 1.16 | -0.01 | 0.639 | 1.575 | 37.2% | 85% |
| emos | 1.30 | -0.07 | 0.714 | 1.664 | 32.9% | 83% |
| trees | 1.10 | -0.04 | 0.601 | 1.509 | 39.6% | 85% |
| model | 1.05 | +0.01 | 0.568 | **1.425** | 41.2% | 83% |

### tmin, lead 2 (10,762 station-days, 216 dates)

| rung | RMSE °C | bias | CRPS | bucket log-loss | modal hit | 80% cover |
|---|---|---|---|---|---|---|
| climatology | 2.83 | +0.14 | 1.547 | 2.410 | 15.4% | 82% |
| persistence | 3.28 | +0.08 | 1.791 | 2.617 | 17.1% | 86% |
| raw_ecmwf | 1.71 | +0.18 | 0.949 | 1.961 | 25.5% | 84% |
| raw_gfs | 2.11 | -0.31 | 1.163 | 2.174 | 21.2% | 86% |
| raw_mean | 1.49 | -0.13 | 0.818 | 1.820 | 29.7% | 85% |
| station_bias | 1.32 | -0.12 | 0.716 | 1.653 | 32.5% | 83% |
| running_bias | 1.26 | -0.01 | 0.694 | 1.657 | 34.0% | 85% |
| emos | 1.39 | -0.04 | 0.762 | 1.730 | 31.1% | 83% |
| trees | 1.21 | -0.07 | 0.663 | 1.606 | 36.2% | 85% |
| model | 1.17 | +0.02 | 0.632 | **1.526** | 37.1% | 83% |

Bucket log-loss: −ln of the probability given to the whole-degree (°C) reading that occurred; lower is better and it is the quantity compared with the market. 80% cover: share of observations inside the central 80% interval; 80% means the stated uncertainty is honest.
