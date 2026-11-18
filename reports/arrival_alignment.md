# Does the market move when the weather centres publish?

Arrivals: `<records>/data/raw/hermes_source_arrivals_2026-11-13/source_arrivals.parquet`. Jumps: forecast-mean changes ≥ 0.3 °C between consecutive scans, daily highs, since 2026-10-30 08:00Z. Hour 0 = the centre's own publication of the ensemble run behind the jump (GEFS last member, or ECMWF ensemble, forecast hour 48). Each cell is the share of the jump the market's implied mean moved in that window; 95% bootstrap over dates; `*` excludes zero.

| lead | jumps | dates | Open-Meteo publishes at | bot sees it at | -6 to -3 h | -3 to -1 h | -1 to +0 h | +0 to +1 h | +1 to +2 h | +2 to +3 h | +3 to +5 h | +5 to +8 h |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 160 | 15 | +1.6 h | +2.0 h | +0.21 (+0.12 to +0.29) * | +0.21 (+0.14 to +0.29) * | +0.14 (+0.08 to +0.21) * | +0.10 (+0.05 to +0.16) * | +0.13 (+0.08 to +0.19) * | +0.06 (+0.02 to +0.10) * | +0.08 (+0.01 to +0.17) * | +0.05 (-0.02 to +0.12) |
| 2 | 73 | 14 | +1.6 h | +1.9 h | +0.30 (+0.03 to +0.64) * | +0.24 (+0.00 to +0.51) * | +0.01 (-0.14 to +0.14) | +0.02 (-0.06 to +0.11) | +0.08 (-0.05 to +0.18) | +0.13 (+0.04 to +0.24) * | +0.15 (+0.01 to +0.30) * | +0.16 (-0.07 to +0.41) |

By model, two days ahead:

| model | jumps | dates | Open-Meteo publishes at | bot sees it at | -6 to -3 h | -3 to -1 h | -1 to +0 h | +0 to +1 h | +1 to +2 h | +2 to +3 h | +3 to +5 h | +5 to +8 h |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| ecmwf_ifs025_ensemble | 26 | 12 | +2.5 h | +3.0 h | +0.48 (-0.07 to +1.05) | +0.15 (-0.24 to +0.73) | +0.04 (-0.17 to +0.36) | -0.25 (-0.58 to +0.05) | -0.06 (-0.50 to +0.21) | +0.10 (-0.06 to +0.21) | +0.10 (-0.20 to +0.36) | +0.39 (-0.24 to +0.89) |
| ncep_gefs025 | 47 | 14 | +1.6 h | +1.9 h | +0.23 (-0.04 to +0.59) | +0.26 (-0.08 to +0.61) | -0.00 (-0.16 to +0.13) | +0.11 (+0.01 to +0.21) * | +0.12 (+0.06 to +0.19) * | +0.14 (+0.03 to +0.28) * | +0.17 (+0.02 to +0.37) * | +0.07 (-0.11 to +0.25) |

Cumulative, all jumps:

| lead | before the centre publishes (−6 to 0 h) | centre publishes → bot sees it | after the bot sees it (to +8 h) |
|---|---|---|---|
| 1 | +0.57 (+0.41 to +0.74) * | +0.24 (+0.16 to +0.33) * | +0.19 (+0.06 to +0.32) * |
| 2 | +0.57 (+0.18 to +1.00) * | +0.07 (-0.09 to +0.25) | +0.42 (+0.04 to +0.85) * |
