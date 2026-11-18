# Author rebuttal: audits of CO-005 (blend) — Argon, Carbon, Astra

- **Author:** Claude Fable (authoring session). **Date:** 2026-11-16. **SOP stage:** 2b (evidence only).
- **Reviewed:** `2026-11-16/Argon/`, `2026-11-16/Carbon/`, `2026-11-16/Astra/`.
- **Code base tested against:** `snapshots/polybot_code_2026-11-14` + CO-005 (NOT Hermes's current code; rerun once `live/code/` is pushed).
- **Evidence:** `evidence/2026-11-16/` (scripts and reports). Argon-section figures came from scratch scripts that were not
  kept; they must be rebuilt as scripts before the master change order cites them.
- This file is the session record of the three cross-examinations, filed unchanged in substance. No self-approval: Chase decides.

## Argon audit: my verdicts (cross-examined 2026-11-16)

| | claim | verdict |
|---|---|---|
| F1 | 20-date stopping rule cannot detect a working strategy | Right in direction. Argon's power numbers 2–3× too pessimistic (assumed 3–8 trades/date; real ≈ 28) |
| F2 | New model's gain over the old pool not shown | Mostly right (finding 11) |
| F3 | "Unseen" dates were used to choose the rule | Right: about 24 replay cells scored on the same 23 dates |
| F4 | `model_lead` docstring: live never older than training | False as I wrote it; Argon right; effect small (finding 13) |
| F5 | Buying at the ask is where the bot loses | Right (finding 10) |
| F6 | Data usage several times my estimate | Right on units (~20 billed units per refresh, ~0.7 M/month; I counted requests). Wrong plan: it is 5 M, not 1 M |
| F7 | `blend_snaps` cannot reconstruct decisions; refresh failures swallowed | Right |
| F8 | blend's baseline covers the wrong markets | Right: `scoring.baselines_forecast` = highs, leads 1–2, all hours, incl. HK/Taipei |
| F9 | Two adverse-selection figures disagree | Unresolved, low impact |

Argon overreach: "a pool nearly always beats its ingredient". At lead 1 both pools lose to the market (+0.04).
Caveat on CO-006: its rule assumes nobody has looked at blend's results; Chase and I already have.
My addition for the merge: report the pays-the-spread score (S3) by bet type.

## My mistakes (acknowledged to Chase)

1. Stopping rule written without a power check.
2. Treated a better log-loss vs the market as a trading edge without checking the retired strategy had the same score.
3. Chose the rule on the dates I called unseen.
4. False freshness claim in `tempmodel.model_lead` and CO-005 §3.1.
5. Understated Open-Meteo usage about 7×.
6. Parity check covered the archive path (`asof=True`), not the live K = 0 path.
7. First model-vs-market result (−0.06, "22 of 23 dates") used loose timing and was wrong; corrected next turn.
8. A failed command left `Oops.rej*` in the project root; removed.
Lesson: before any strategy change order, run the bet-type breakdown and the same-events old-vs-new comparison.

## Carbon audit cross-examination (added 2026-11-16, after Argon)

Files: `change_orders/2026-11-16/Carbon/` (audit, CO-006 doc, CO-006.diff), `POLYBOT_MULTI_MODEL_AUDIT_SOP.md`.
Evidence: `pipeline/research_gate_fees_order.py`, `data/reports/gate_fees_order.md`.

**Patch:** applies at `--fuzz=0` on snapshot + CO-005 and compiles. DEFECT: its `PREREGISTRATION.md` hunk (`@@ -1,96 +1,19 @@`)
deletes the title and sections 1-6 and leaves only section 7. Do not apply as written. Tested against the 11-14 snapshot, not Hermes's current code.

**Confirmed (code or data):**
- Fees: `fee_rate` 0.05 on 69.6% of lead-2 scans, every date from 2026-11-03; forecast paid $14.16. My replay assumed zero.
  With real fees the deployed rule replays at -0.7% (-7.8 to +6.9); ignoring fees +1.8%.
- Brier: pool minus market +0.00 (-0.00 to +0.01), 12 of 23 dates; log-loss -0.04 significant. Edge is in the tails only.
- 15:00 gate deployed vs >=18:00 tested (via `PEAK_HOUR`; there was no `blend_min_local_hour` key in CO-005).
- Ascending bucket order crowds the $4 event cap; dead zone 0.7843 < ask <= 0.90; reject spam; ex-post max(B1,B2) baseline;
  proxy stations in the export; missing log columns; `inputs()` clock not injectable.

**Not supported / wrong:**
- 18:00 gate has no measurable benefit: +$5 (-49 to +52). Entries at 15-18h did better (+4.8%) than >=18h (-7.7%), neither significant.
- Price cap 0.78 changes replay P/L by exactly $0 (those orders could never fill). It only stops reject spam.
- Claimed +$82.79 improvement: my paired replay gives +$43 (-9 to +90) with fresh inputs, +$77 (+5 to +143) if 15-18h inputs were a day stale.
  Almost all of it is the edge sort: +$38 (-3 to +74). In-sample, same 23 dates.
- Hallucinated names: config keys `live`, `forecast_Order`; `dashboard._latest_books()`; `strategy_epoch_ts`. `baselines_forecast` is highs only.
- Dashboard and `fill_realism` diagnosis likely backwards (`snapshots` and `follow_snaps` are still written).
- Plan is Professional 5 M calls, not Standard 1 M. `var_scale` 3.79 is mostly the Jensen correction (~3.56).

**Best variant under real fees:** 18:00 + edge sort + cap = +2.5% (-4.9 to +11.0). Expected win rate 66%, realised 56%. No variant excludes zero.

**SOP line "Do not defend prior design choices":** recommended replacing with an author-rebuttal stage (author may contest only with
executable evidence; data-confirmed findings stand; author never approves own work). Awaiting Chase's decision.

**Still HOLD on the master change order.**

## Astra audit cross-examination (added 2026-11-16, after Carbon)

Files: `change_orders/2026-11-16/Astra/` (audit, amendment requests, `verify_astra.py`, `verify_integrity.py`, `audit_archived_ledger.py`, `results/`).
Evidence: `pipeline/research_settlement_and_maker_checks.py`, `data/reports/settlement_and_maker_checks.md`.
Reproduced locally on snapshot + CO-005 + Carbon CO-006: 19 tests, 12 pass, 7 fail, exactly as Astra reports. Integrity and ledger scripts also match.
Astra wrote no diff, only amendment requests. It was not a blind review (it read Carbon first, and says so). SOP edited to v1.1 the same day.

| ID | Astra says | Verdict | Evidence |
|---|---|---|---|
| A1 | CO-006 deletes pre-registration | CONFIRMED, blocker | same finding as mine |
| A2 | `go_live()` ignores new section 7 gate | CONFIRMED, must fix in master CO | fixture ready=True vs PENDING; READY is a prompt to Chase, live mode not implemented |
| A3 | cap/cash checks exclude fee | CONFIRMED, severity overstated (P0 -> P2) | my CO-005 code; overshoot <= 5% of stake, max $0.0994 per fill in archive; lowest cash ever $50.61 |
| A4 | settles on `closed` with empty UMA status, winner by price > 0.5 | Code path CONFIRMED, never occurred | 2,224 fully resolved events all have exactly one YES; 0 of 24,493 markets resolved before day end; 99.76% agree with METAR off proxy stations |
| A5 | settle sweep ignores `blend_snaps` | CONFIRMED latent | covered today only because retired `strategy.py` still writes `snapshots` (same leads 1-2) |
| A6 | maker rule ignores taker direction | CONTESTED, bounded | 51 of 447 maker fills (12% of shares) depend on it; retired path; tape is 86% BUY-labelled |
| A7 | future cache timestamp accepted | CONFIRMED, test-only path | `now` argument is added by Carbon's patch |
| A8 | refresh change alters epoch hash | CONFIRMED | matches my Carbon item 11 |
| A9 | partial fill recorded as `filled` | CONFIRMED, cosmetic | `settle.py` already accepts `'partial'`, broker never writes it |

Caution on A4's fix: the archive does not record `umaResolutionStatus`, so it is unknown whether weather markets normally populate it.
Requiring "resolved" could stall all settlement. Safer guard: pay only when the price vector is exactly 0/1; get captured Gamma responses from Hermes first.

**Still HOLD on the master change order.** Inputs now in hand: Argon, Carbon, Astra, my rebuttals. Still missing: Hermes's current code.
