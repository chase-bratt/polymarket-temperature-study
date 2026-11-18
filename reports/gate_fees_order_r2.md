<!-- Supersedes evidence/2026-11-16/gate_fees_order.md: adds section D (the cap arithmetic the bot actually uses, and Carbon Rev B). Sections A to C are unchanged in method. -->
# Blend rule: fees, Brier, entry gate, ordering, price cap (2026-11-16)

Output of `pipeline/research_gate_fees_order.py`. 23 market dates, daily highs, lead 2, real fee rates, date bootstrap. Section D uses the cap arithmetic of the bot code; section C is the first version, which counted stakes only.

```
== A. fees ==
 fee_rate      n         d0         d1  pct
     0.00 153681 2026-10-23 2026-11-02 30.4
     0.05 351208 2026-10-20 2026-11-15 69.6
         d  avg_fee  share_fee
2026-10-20     0.05        1.0
2026-10-21     0.05        1.0
2026-10-22     0.05        1.0
2026-10-23     0.00        0.0
2026-10-24     0.00        0.0
2026-10-25     0.00        0.0
2026-10-26     0.00        0.0
2026-10-27     0.00        0.0
2026-10-28     0.00        0.0
2026-10-29     0.00        0.0
2026-10-30     0.00        0.0
2026-10-31     0.00        0.0
2026-11-01     0.00        0.0
2026-11-02     0.00        0.0
2026-11-03     0.05        1.0
2026-11-04     0.05        1.0
2026-11-05     0.05        1.0
2026-11-06     0.05        1.0
2026-11-07     0.05        1.0
2026-11-08     0.05        1.0
2026-11-09     0.05        1.0
2026-11-10     0.05        1.0
2026-11-11     0.05        1.0
2026-11-12     0.05        1.0
2026-11-13     0.05        1.0
2026-11-14     0.05        1.0
2026-11-15     0.05        1.0
strategy  fills  fees    cost  paid         d0         d1  first_fee
forecast    834 14.16 2222.70   208 2026-10-18 2026-11-13 2026-10-18
  follow     49  0.00  244.42     0 2026-10-21 2026-11-08        NaT
intraday     35  0.00  100.11     0 2026-10-23 2026-10-25        NaT

== B. multiclass Brier and log-loss, lead 2 (pool - market, date bootstrap) ==
late: events 940 dates 23  Brier market 0.6877 model 0.7267 pool 0.6883  pool-market +0.00 (-0.00 to +0.01)  dates pool better 12/23  | logloss pool-market -0.04 (-0.05 to -0.02) *
strict: events 978 dates 23  Brier market 0.7050 model 0.7538 pool 0.7064  pool-market +0.00 (-0.00 to +0.01)  dates pool better 11/23  | logloss pool-market -0.04 (-0.06 to -0.03) *

== C. replay of the bot rule on 23 market dates, lead 2, with real fee rates ==
v1 15:00 gate, asc, px<=0.90, fresh inputs n  514 dates 23 win 54% exp 65% px 0.54 fees $ 17.07 P/L $  -10.99 ROI -0.7% (-7.8 to +6.9)  winning dates 9/23
v1 15:00 gate, asc, px<=0.90, stale 15-18h n  600 dates 23 win 54% exp 66% px 0.55 fees $ 19.87 P/L $  -45.31 ROI -2.6% (-9.4 to +4.4)  winning dates 7/23
v1 same (fresh), fees ignored              n  574 dates 23 win 55% exp 65% px 0.54 fees $  0.00 P/L $  +29.25 ROI +1.8% (-4.9 to +9.0)  winning dates 10/23
18:00 gate only (asc, 0.90)                n  448 dates 23 win 53% exp 64% px 0.53 fees $ 14.90 P/L $   -5.98 ROI -0.5% (-7.9 to +8.1)  winning dates 7/23
18:00 gate + edge sort (0.90)              n  446 dates 23 win 56% exp 66% px 0.55 fees $ 14.77 P/L $  +31.77 ROI +2.5% (-5.0 to +11.0)  winning dates 10/23
18:00 gate + px<=0.78 (asc)                n  448 dates 23 win 53% exp 64% px 0.53 fees $ 14.90 P/L $   -5.98 ROI -0.5% (-7.9 to +7.7)  winning dates 7/23
v2 Carbon: 18:00 + edge sort + px<=0.78    n  446 dates 23 win 56% exp 66% px 0.55 fees $ 14.77 P/L $  +31.77 ROI +2.5% (-4.9 to +11.0)  winning dates 10/23
15:00 gate + edge sort + 0.78 (fresh)      n  512 dates 23 win 56% exp 66% px 0.55 fees $ 16.99 P/L $   -3.43 ROI -0.2% (-7.4 to +8.0)  winning dates 9/23
  paired v2 minus v1(fresh): total diff $+42.75 (95% -9.31 to +89.53), dates better 16/23
  paired v2 minus v1(stale): total diff $+77.08 (95% +5.41 to +143.30), dates better 17/23
  paired 18:00 gate only minus v1(fresh): total diff $+5.01 (95% -48.96 to +52.48), dates better 13/23
  paired edge sort effect at 18:00: total diff $+37.75 (95% -2.92 to +74.26), dates better 12/23
  paired price cap effect at 18:00: total diff $+0.00 (95% +0.00 to +0.00), dates better 0/23

== D. same replay with the cap arithmetic the bot actually uses (fees count toward the $4 event cap) ==
CO-005 live rule, fresh inputs             n  511 dates 23 win 54% exp 65% px 0.54 fees $ 16.80 P/L $   -4.71 ROI -0.3% (-7.6 to +7.9)  winning dates 9/23  events 506 max/event 2
CO-005 live rule, stale 15-18h             n  597 dates 23 win 54% exp 67% px 0.55 fees $ 19.60 P/L $  -39.04 ROI -2.2% (-9.0 to +4.6)  winning dates 7/23  events 592 max/event 2
Carbon Rev A (18:00, edge sort, 0.78)      n  443 dates 23 win 56% exp 66% px 0.55 fees $ 14.51 P/L $  +31.12 ROI +2.4% (-5.1 to +11.0)  winning dates 10/23  events 440 max/event 2
Carbon Rev B (18:00, edge sort, 0.77, fee-inclusive) n  442 dates 23 win 56% exp 66% px 0.55 fees $ 14.48 P/L $  +30.34 ROI +2.4% (-5.6 to +10.9)  winning dates 10/23  events 439 max/event 2
Rev B rule but 15:00 gate (fresh)          n  508 dates 23 win 56% exp 67% px 0.55 fees $ 16.70 P/L $   +8.12 ROI +0.5% (-6.6 to +8.9)  winning dates 9/23  events 505 max/event 2
  paired Rev B minus CO-005 (fresh): total diff $+35.06 (95% -21.45 to +84.88), dates better 16/23
  paired Rev B minus CO-005 (stale): total diff $+69.38 (95% -3.53 to +137.29), dates better 16/23
  paired Rev B minus Rev A: total diff $-0.78 (95% -2.34 to +0.00), dates better 0/23
  paired 18:00 minus 15:00 under Rev B rule: total diff $+22.23 (95% -14.98 to +58.48), dates better 14/23

== entries by hour window, v1 fresh ==
entered >=18h                              n  228 dates 23 win 50% exp 65% px 0.55 fees $  8.28 P/L $  -50.70 ROI -7.7% (-19.0 to +4.5)  winning dates 5/23
entered 15-18h                             n  286 dates 21 win 57% exp 65% px 0.53 fees $  8.79 P/L $  +39.71 ROI +4.8% (-5.1 to +15.0)  winning dates 11/21

== v1 fresh by price band ==
ask 0.05-0.30                              n   36 dates 16 win 17% exp 26% px 0.15 fees $  1.17 P/L $  +13.06 ROI +17.8% (-58.8 to +113.4)  winning dates 5/16
ask 0.30-0.60                              n  311 dates 23 win 51% exp 63% px 0.51 fees $ 10.71 P/L $  -21.55 ROI -2.6% (-11.7 to +7.8)  winning dates 10/23
ask 0.60-0.78                              n  166 dates 23 win 68% exp 78% px 0.68 fees $  5.09 P/L $   -0.40 ROI -0.1% (-10.8 to +10.3)  winning dates 13/23
dead zone 0.7843<ask<=0.90: 30 distinct market-sides qualified but could never be bought; they would have won 90% at avg ask 0.82

== side mix ==
v1 15:00 gate, asc, px<=0.90, fresh inputs {'NO': {'n': 472, 'pnl': -24.02, 'win': 0.57}, 'YES': {'n': 42, 'pnl': 13.03, 'win': 0.19}}
v2 Carbon: 18:00 + edge sort + px<=0.78 {'NO': {'n': 418, 'pnl': 17.66, 'win': 0.58}, 'YES': {'n': 28, 'pnl': 14.1, 'win': 0.25}}
```
