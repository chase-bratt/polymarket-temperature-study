# Does the blend's edge survive real prices and fills? (daily highs, market dates)

Tuned temperature model fitted to 2026-10-09; log pool with the market at w = 0.5 (not tuned). One share per position, held to resolution, no fees. 95% bootstrap over whole dates; `*` excludes zero.

## Timing variants: log-loss on the winning bucket

| timing | days ahead | events | dates | market | model alone | pool | pool − market (95%) | dates pool beats market |
|---|---|---|---|---|---|---|---|---|
| strict | 1 | 1,102 | 25 | 1.275 | 1.487 | 1.317 | +0.04 (+0.03 to +0.06) * | 3 of 25 |
| strict | 2 | 978 | 23 | 1.504 | 1.592 | 1.462 | -0.04 (-0.06 to -0.02) * | 19 of 23 |
| late | 1 | 997 | 25 | 1.237 | 1.392 | 1.263 | +0.03 (+0.00 to +0.05) * | 9 of 25 |
| late | 2 | 940 | 23 | 1.429 | 1.482 | 1.388 | -0.04 (-0.05 to -0.03) * | 20 of 23 |
| loose | 1 | 1,102 | 25 | 1.275 | 1.367 | 1.269 | -0.01 (-0.03 to +0.01) | 12 of 25 |
| loose | 2 | 978 | 23 | 1.504 | 1.478 | 1.410 | -0.09 (-0.11 to -0.08) * | 22 of 23 |

## Trading replay, `strict` timing

| days ahead | execution | edge threshold | positions | dates | won vs price paid | return per $ (95%) | P/L, 1 share each |
|---|---|---|---|---|---|---|---|
| 2 | taker | 0.05 | 1826 | 23 | 51% vs 53% | -3.1% (-6.9 to +0.5) | $-30.13 |
| 2 | taker | 0.08 | 1088 | 23 | 53% vs 55% | -2.4% (-7.1 to +2.1) | $-14.46 |
| 2 | taker | 0.12 | 478 | 23 | 55% vs 56% | -1.1% (-7.0 to +4.8) | $-2.82 |
| 2 | maker (1611/3545 filled) | 0.05 | 1611 | 23 | 49% vs 50% | -2.9% (-7.0 to +1.1) | $-23.83 |
| 2 | maker (957/2345 filled) | 0.08 | 957 | 23 | 52% vs 53% | -1.8% (-7.4 to +3.2) | $-9.18 |
| 2 | maker (401/1329 filled) | 0.12 | 401 | 22 | 52% vs 55% | -6.1% (-13.9 to +1.7) | $-13.37 |
| 1 | taker | 0.05 | 2718 | 25 | 47% vs 47% | -0.8% (-3.4 to +1.9) | $-10.00 |
| 1 | taker | 0.08 | 1553 | 25 | 50% vs 48% | +3.4% (-0.4 to +7.7) | $+25.60 |
| 1 | taker | 0.12 | 534 | 25 | 51% vs 49% | +5.2% (-3.9 to +15.1) | $+13.69 |
| 1 | maker (2592/3445 filled) | 0.05 | 2592 | 24 | 44% vs 46% | -3.5% (-6.4 to -0.7) * | $-40.96 |
| 1 | maker (1498/2083 filled) | 0.08 | 1498 | 24 | 47% vs 47% | +0.2% (-4.3 to +4.5) | $+1.56 |
| 1 | maker (555/846 filled) | 0.12 | 555 | 23 | 48% vs 47% | +1.3% (-5.2 to +8.7) | $+3.46 |

## Trading replay, `late` timing

| days ahead | execution | edge threshold | positions | dates | won vs price paid | return per $ (95%) | P/L, 1 share each |
|---|---|---|---|---|---|---|---|
| 2 | taker | 0.05 | 1351 | 23 | 52% vs 51% | +1.3% (-2.8 to +5.4) | $+9.07 |
| 2 | taker | 0.08 | 648 | 23 | 54% vs 52% | +3.9% (-2.7 to +11.0) | $+13.17 |
| 2 | taker | 0.12 | 203 | 22 | 57% vs 53% | +7.1% (-8.1 to +22.2) | $+7.68 |
| 2 | maker (1047/2278 filled) | 0.05 | 1047 | 23 | 47% vs 49% | -3.1% (-8.6 to +2.1) | $-15.87 |
| 2 | maker (529/1247 filled) | 0.08 | 529 | 23 | 50% vs 50% | +1.4% (-7.2 to +8.5) | $+3.68 |
| 2 | maker (146/492 filled) | 0.12 | 146 | 22 | 55% vs 51% | +7.6% (-7.2 to +23.1) | $+5.63 |
| 1 | taker | 0.05 | 1574 | 25 | 44% vs 45% | -2.3% (-7.5 to +3.0) | $-16.42 |
| 1 | taker | 0.08 | 725 | 25 | 44% vs 46% | -3.9% (-11.6 to +4.5) | $-12.90 |
| 1 | taker | 0.12 | 199 | 25 | 45% vs 47% | -4.4% (-19.0 to +11.6) | $-4.14 |
| 1 | maker (1455/2071 filled) | 0.05 | 1455 | 24 | 42% vs 44% | -3.8% (-8.7 to +1.0) | $-24.47 |
| 1 | maker (711/1045 filled) | 0.08 | 711 | 24 | 40% vs 44% | -9.3% (-19.0 to -0.1) * | $-29.45 |
| 1 | maker (220/331 filled) | 0.12 | 220 | 24 | 39% vs 44% | -11.7% (-26.1 to +4.8) | $-11.24 |
