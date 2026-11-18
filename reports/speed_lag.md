# Speed-lag test: daily highs, one and two days ahead

Window: scans and trades since 2026-10-30 08:00Z (model-run-triggered refresh live). Intervals are 95% bootstraps over whole target dates; `*` marks an interval that excludes zero.

## T1. Follow-through after a forecast jump (≥ 0.3 °C between consecutive scans)

Each cell: share of the forecast jump that the market's implied mean moved in that window (1.00 = moved as much as the forecast).

| lead | jumps | dates | avg jump °C | 6 h before → scan before | 3 h before → scan before | scan before → this scan | this scan → +1 h | → +3 h | → +6 h | → +12 h |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 162 | 15 | 0.42 | +0.68 (+0.50 to +0.90) * | +0.41 (+0.26 to +0.57) * | +0.01 (-0.01 to +0.03) | +0.04 (+0.00 to +0.08) * | +0.09 (+0.01 to +0.20) * | +0.14 (+0.02 to +0.27) * | +0.08 (-0.08 to +0.29) |
| 2 | 75 | 14 | 0.41 | +0.59 (+0.25 to +0.94) * | +0.06 (-0.16 to +0.29) | +0.03 (+0.00 to +0.06) * | +0.13 (+0.04 to +0.21) * | +0.28 (+0.08 to +0.47) * | +0.40 (+0.03 to +0.77) * | +0.54 (+0.13 to +0.97) * |

## T2. Are real takers paid more just after a new run?

Every captured print at 15–85¢ on highs, held to resolution, from the taker's side.

| lead | time since the newest ECMWF-ENS / GEFS run | $ traded | taker return (95%) | dates |
|---|---|---|---|---|
| 1 | 0–1 h after a new run | 490,572 | +0.4% (-3.4 to +4.2) | 13 |
| 1 | 1–3 h | 730,931 | +2.3% (-1.1 to +5.9) | 13 |
| 1 | 3–6 h | 331,961 | +3.7% (+0.5 to +7.4) * | 13 |
| 1 | 6 h+ | 0 | – | 0 |
| 1 | **all** | 1,553,463 | +2.0% (-0.4 to +4.3) | 13 |
| 2 | 0–1 h after a new run | 95,776 | +4.0% (-0.2 to +8.1) | 12 |
| 2 | 1–3 h | 163,820 | +4.4% (+0.4 to +9.2) * | 12 |
| 2 | 3–6 h | 90,438 | +2.8% (-4.5 to +9.4) | 12 |
| 2 | 6 h+ | 0 | – | 0 |
| 2 | **all** | 350,034 | +3.9% (+1.9 to +6.4) * | 12 |

## T3. Tradable replay: react to the jump as a taker

At the scan that saw the jump, buy the bucket whose model probability rose most (YES at the ask) or fell most (NO at the ask), only if it changed by ≥ 5 points. Price columns are in dollars per share, after paying the full spread on both ends.

| lead | leg | trades | dates | avg ask paid | spread paid | bid 6 h later − ask paid | bid 12 h later − ask paid | hold to resolution |
|---|---|---|---|---|---|---|---|---|
| 1 | down: buy NO | 59 | 15 | 0.75 | 0.030 | -0.02 (-0.05 to -0.00) * | -0.06 (-0.10 to -0.03) * | +0.4% (-19.0 to +18.6) |
| 1 | up: buy YES | 126 | 15 | 0.30 | 0.034 | -0.03 (-0.05 to -0.01) * | -0.03 (-0.05 to -0.00) * | -14.3% (-37.3 to +11.3) |
| 2 | down: buy NO | 28 | 14 | 0.77 | 0.081 | -0.06 (-0.10 to -0.03) * | -0.06 (-0.10 to -0.02) * | +2.8% (-17.7 to +19.5) |
| 2 | up: buy YES | 45 | 14 | 0.30 | 0.054 | -0.03 (-0.04 to -0.02) * | -0.03 (-0.05 to -0.01) * | -1.2% (-23.5 to +23.2) |

## How late the bot's data source is

| model | runs | hours from model start to Open-Meteo availability (median) | 90th pct |
|---|---|---|---|
| ncep_nbm_conus | 430 | 1.0 | 1.4 |
| ncep_hrrr_conus | 480 | 1.6 | 2.0 |
| dwd_icon | 86 | 3.6 | 4.6 |
| ecmwf_aifs025_single | 86 | 5.7 | 7.6 |
| ncep_gfs025 | 83 | 5.9 | 7.1 |
| ukmo_global_deterministic_10km | 86 | 7.5 | 8.8 |
| ecmwf_ifs025 | 86 | 7.7 | 9.1 |
