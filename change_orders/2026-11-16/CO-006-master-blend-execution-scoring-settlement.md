# CO-006 (master): blend execution, evaluation rule, and settlement safety

| | |
|---|---|
| Status | **APPROVED FOR HERMES** by Chase in chat, 2026-11-16 ("Push all the changes that will improve the model, dont wait"). Verified on the 2026-11-14 snapshot; the check against Hermes's current code is step 1 of section 6 and stops the apply if it fails. |
| Author | Claude Fable 5.2, on Chase's instruction of 2026-11-16 ("Yes, write it") |
| Supersedes | `2026-11-16/Argon/CO-006…` and `2026-11-16/Carbon/CO-006…` (Rev B). Both stay on file, unimplemented. |
| Diff | `CO-006.diff`, sha256 `817c1ef42d774db4ff2434a1d591e74bba48155ffe7746fb99850cb361423212`, 9 files |
| Base | `snapshots/polybot_code_2026-11-14` + `2026-11-14/CO-005.diff` (sha256 `58140cd9…`). `live/` had no push when this was written. |
| Tests | `verify_master.py` (24 tests) and Astra's `verify_astra.py` (19 tests) |
| Rules kept | Paper only. No cap raised. No ledger history deleted. Retired strategies stay retired. |

**Hermes: your instructions are section 6. Run it start to finish. Sections 1 to 5 explain what the patch does;
section 7 is the rollback.**

## 1. What this is

Three audits of CO-005 (Argon, Carbon, Astra) and my rebuttals agree on what is wrong with `blend` as deployed.
This order takes Carbon's Rev B patch as the base, because it was verified line by line, and adds the fixes Astra
asked for that Carbon left out of scope, plus four defects I found in Rev B while reviewing it for this order.

```
 Argon audit ─┐
 Carbon Rev B ─┼─► cross-examined against code and data ─► master CO-006 (this file)
 Astra audit ─┘                                              │
                                                             ▼
        ┌────────────────────────────┬────────────────────────────┬──────────────────────────┐
        │ A. how blend picks trades  │ B. how blend is judged     │ C. how positions settle  │
        │ gate 18:00, best edge      │ one baseline, four scores, │ pay only on a final 0/1  │
        │ first, fees inside caps    │ 30 and 60 date rule        │ result, atomically       │
        └────────────────────────────┴────────────────────────────┴──────────────────────────┘
                                                             │
                                                             ▼
                     Hermes dry-run on live code ─► Chase approves ─► apply ─► register ─► verify
```

## 2. Changes

### A. Execution (`polybot/blend.py`, `config.py`, `config.json`)

| # | Change | From | Why |
|---|---|---|---|
| A1 | Lead-2 markets trade only from station-local 18:00 (`blend_min_local_hour = 18.0`). CO-005 opened at 15:00. | Carbon | CO-005 cited a replay that tested 18:00 onward. My mistake. |
| A2 | Candidates in an event are taken in descending net-edge order, not ascending bucket order. | Carbon | Fees count toward the $4 cap, so an event gets one position (511 positions in 506 events in replay). Order decides which trade is made. |
| A3 | `blend_max_price` 0.90 to 0.77. Event cap, open cap and cash checks include the fee. | Carbon, Astra A3 | Orders above 0.784 could never fill and wrote a rejection every scan. At 0.78 the fee pushed exposure to $4.02 over a $4 cap. |
| A4 | Held, opposite-held and event-capped candidates are skipped without a rejected order row. | Carbon, Argon | Thousands of synthetic rejections in `orders`. |
| A5 | 11 new `blend_snaps` columns: input age, local hour, hours to peak, tradable flag, both edges, fee rate, both ask depths, model name, version. | Argon, Carbon | Needed to score blend with exact timing. Added by `ALTER TABLE` on first scan; old rows keep NULL. |
| A6 | A failed input refresh is reported in the scan output and the run note (`blend_rf=`). | Argon, Carbon | Stale inputs were silent. |
| A7 | A cached input stamped after "now" is never fresh and never tradable. | Astra A7 | Guards the injectable clock that A5 introduces. |
| A8 | `blend_version` to `2026-11-16-v2`. | Carbon | Entry rules change, so the scoring clock restarts. |

### B. Evaluation (`polybot/scoring.py`, `tools/scorecard.py`, `notes/PREREGISTRATION.md` section 7)

| # | Change | From | Why |
|---|---|---|---|
| B1 | One baseline fixed in advance, `B2_blend`: favourite NO at ask plus fee on blend's own scans. | Carbon | CO-005 compared against the better of two baselines, chosen after the fact, on the wrong universe. |
| B2 | Four scores on each event's first tradable scan: S1a log-loss vs market, S1b Brier vs market, S2 log-loss vs the retired forecast pool, S3 net return per share of every qualifying signal. | Carbon | Log-loss alone flattered the blend: its Brier score is no better than the market's. |
| B3 | Rule: PENDING under 30 effective dates. From 30, retire if S1a mean >= 0 or the whole S3 band is below zero. From 60, PASS needs all four. Otherwise NOT PASSED: stays paper, Chase decides. | Argon F1, Carbon | My 20-date rule passed a real +7% edge only 31% of the time. |
| B4 | `go_live` reports READY for blend only with a section 7 PASS, and for any strategy only with zero integrity flags. | Astra A2 | Rev A could print READY while its own evaluation said PENDING. |
| B5 | `fill_realism` reads `blend_snaps` for blend. | Carbon | Decoupling. Not a bug fix: the old table still had the rows. |
| B6 | Section 7 appended to the pre-registration. Sections 1 to 6 untouched. Marked "pending approval by Chase". | Carbon, Astra A1 | Carbon's first diff deleted sections 1 to 6. |

### C. Settlement (`polybot/settle.py`)

| # | Change | From | Why |
|---|---|---|---|
| C1 | A closed market pays only when its price vector is final: exactly 1 on one side and 0 on the other. Anything else waits and is recorded in `settle_skips` (one row per market, deleted when it settles). A market waiting more than two days is called out for manual review in the cycle log. | Astra A4 | The old code picked the winner by price > 0.5 on any closed market. |
| C2 | The settlement sweep also covers markets that appear only in `blend_snaps`. | Astra A5 | Blend learned outcomes only because the retired forecast strategy still writes `snapshots`. Switching that off would have silently stopped blend's scoring. |
| C3 | Price-history capture runs after the positions and the market row are written. | Found in review | Capture commits internally. Before, a crash right after it left the market marked resolved with its position open, and that market was never swept again. |

C1 does not require `umaResolutionStatus`. Nobody has confirmed weather markets fill that field in, so requiring it
could freeze every settlement. A status that is present and not "resolved" still blocks, as before.

Known limit of C1: a market that resolves as a split (0.5 / 0.5) is not paid by the bot. It waits in `settle_skips`
and its position stays open, counting toward the caps, until Chase decides how to book it. The old code paid such a
market as a NO win. No split has occurred in the 2,224 events resolved so far.

### What I changed in Carbon's Rev B

| Defect in Rev B | Fix |
|---|---|
| The 30-date check in code retired blend unless S1a was already significant and S3's mean was positive. The pre-registration text said something milder. Code and text disagreed. | Code now follows the text, and both are tested (`verify_master.py`). |
| At 60 dates the code printed RETIRE for any miss. The text said retire only if the S3 band is below zero. | Third outcome added: NOT PASSED. |
| The text defined S3 over settled positions. The code computes it over signals. | Text now describes what the code does. |
| Evaluation ran one query per event against a table with no index on `event_id` (about 47,000 new rows a day). | One pass, no sort, no per-event query. |
| A docstring and a config comment said 18:00 local is "after 12Z runs". True in the Americas, false in Asia. The docstring also carried three numbers I could not verify. | Reworded to what is verified. |

## 3. What is left out, and why

| Item | Why not now |
|---|---|
| Astra A6, maker fills ignore trade direction | Contested, and the maker path is retired. Worst case it overstates past maker fills by 12% (51 of 447). Needs captured examples of the tape's direction labels first. `verify_astra.py` still fails this one test, on purpose. |
| Astra A9, partial fills labelled "filled" | Cosmetic. Quantities and cash reconcile. |
| Dashboard | Hermes changed it after the snapshot. Carbon's Rev B also dropped it. |
| Changes to the temperature model itself | Tested on 2026-11-16 against a rule set beforehand; none passed. See the table below. |
| Any change to stake, caps or the 0.08 edge threshold | No evidence supports one, and caps are never raised. |

Model ideas tested and not shipped (`evidence/2026-11-16/model_variants.md`). The rule, fixed before running: ship
only what improves the test-split score with a 95% interval excluding zero and does not worsen the blend's scores
against the market.

| Idea | Result | Verdict |
|---|---|---|
| Train without the two proxy stations (Hong Kong, Taipei) | No difference on market dates: score change 0.00 (−0.01 to +0.00), error 1.121 vs 1.119 °C | No gain either way. Package unchanged. |
| Heavier tails (Student-t instead of the normal curve) | Better on the test split (−0.02, significant), but the blend then does worse against the market (S1a +0.005, significant) and signal return falls from 2.5c to 1.7c per share | Fails the rule. Not shipped. |
| Resize the predicted spread | The spread is already slightly too wide, not too narrow: 83 to 85% of outcomes land in the 80% band | Nothing to fix. The blend's overconfidence comes from which trades it selects, not from the model's spread. |

## 4. What to expect

Replay of the exact rule on the 23 archived market dates, lead 2, real fee rates
(`evidence/2026-11-16/gate_fees_order_r2.md`, section D):

| Rule | Return on stake (95% range) | Profit, 23 dates |
|---|---|---|
| CO-005 as deployed, newest inputs | −0.3% (−7.6 to +7.9) | −$4.71 |
| CO-005 as deployed, day-old inputs from 15:00 to 18:00 | −2.2% (−9.0 to +4.6) | −$39.04 |
| This order (execution identical to Carbon Rev B) | +2.4% (−5.6 to +10.9) | +$30.34 |

The improvement over CO-005 is +$35 to +$69 depending on the input assumption. Neither range excludes zero, and the
same 23 dates were used to find the problems. My estimate that `blend` clears its costs stays at 10 to 15%.

How the section 7 rule behaves, simulated from those dates with the bot's own scoring functions
(`evidence/2026-11-16/stopping_rule_power.md`):

| True edge per share | Retired at 30 dates | Retired at 60 | Not passed at 60 | PASS |
|---|---|---|---|---|
| None, and no forecasting edge either | 50% | 12% | 38% | 0% |
| 0c | 1% | 0% | 98% | 1% |
| +2c | 0% | 0% | 92% | 8% |
| +4c | 0% | 0% | 65% | 35% |
| +6c | 0% | 0% | 24% | 76% |
| +8c | 0% | 0% | 4% | 96% |

The replays put the edge between 0c and +2c per share, so the most likely result after 60 dates is NOT PASSED.
That is the rule working: it will not call a small or absent edge a pass.

## 5. Effect on the ledger and epochs

- `blend_version` changes, so `blend` starts a new epoch at the first scan after apply. The 40 open positions from the
  first epoch settle normally and are scored in that epoch. They still count toward the $200 open cap and block a
  second position in their markets.
- `blend_snaps` gains 11 nullable columns. A new table `settle_skips` holds at most one row per unsettled market.
- No row is deleted or rewritten anywhere.
- `scoring.py`, `tools/scorecard.py` and `PREREGISTRATION.md` are locked files, so the manifest must be re-registered.

## 6. Instructions for Hermes

Chase approved this order on 2026-11-16. **Run steps 0 to 8 in order, in one session, without asking for a further
go-ahead.** Stop only where a step says STOP; then change nothing more and report to Chase what you saw.

Ground rules for this job:

- Paper only. Do not raise any cap. Do not delete or rewrite ledger rows.
- Never force the patch: `--fuzz=0` always, no hand-editing of hunks, no `--force`.
- Do only what these steps say. If something unexpected happens that no step covers, STOP and report.

**Step 0. Put the patch in place and check it is the right file.** Copy `CO-006.diff` to `/opt/data/CO-006.diff`.

```bash
sha256sum /opt/data/CO-006.diff
```

The hash must be `817c1ef42d774db4ff2434a1d591e74bba48155ffe7746fb99850cb361423212`. If not: STOP.

**Step 1. Dry run on your current code.** This changes nothing.

```bash
cd /opt/data && patch -p1 --fuzz=0 --dry-run < CO-006.diff
```

Every one of the 9 files must check clean. If any hunk fails: STOP, tell Chase which file failed, and send him a fresh
code export (`tools/export_code.py`). The patch was built against the 2026-11-14 export plus CO-005, so a failure means
that file has changed since.

**Step 2. Disk space, then back up the nine files** (not the database; nothing in it is rewritten).

```bash
df -h /opt/data
```

If less than 100 MB is available: STOP.

```bash
cd /opt/data && tar czf polybot/data/pre_CO-006_files.tgz polybot/config.json polybot/notes/PREREGISTRATION.md polybot/polybot/__main__.py polybot/polybot/blend.py polybot/polybot/config.py polybot/polybot/scoring.py polybot/polybot/settle.py polybot/polybot/tempmodel.py polybot/tools/scorecard.py
```

**Step 3. Check what Polymarket returns for markets that already settled.** The new settlement rule pays only when the
price vector is exactly 1 and 0. This confirms that is what settled markets look like. It changes nothing.

```bash
cd /opt/data/polybot && .venv/bin/python -c "
import json
from polybot import db, gamma
con = db.connect()
bad = n = 0
for r in con.execute('SELECT market_id, outcome FROM markets WHERE resolved=1 ORDER BY resolved_at DESC LIMIT 12'):
    m = gamma.fetch_market(r['market_id']); n += 1
    raw = m.get('outcomePrices')
    try:
        pr = [float(x) for x in (json.loads(raw) if isinstance(raw, str) else raw)]
    except Exception:
        pr = None
    final = 'YES' if pr == [1.0, 0.0] else 'NO' if pr == [0.0, 1.0] else None
    ok = bool(m.get('closed')) and final == r['outcome']
    bad += not ok
    print('ok  ' if ok else 'BAD ', r['market_id'], 'closed', m.get('closed'), 'uma', repr(m.get('umaResolutionStatus')), 'prices', raw, 'ledger', r['outcome'])
print('RESULT:', 'OK' if n and not bad else 'STOP', f'({n} checked, {bad} bad)')
"
```

If the last line is `RESULT: STOP`: STOP, and paste the whole output to Chase. Do not apply. If it is `RESULT: OK`,
keep the output for the report and continue.

**Step 4. Apply and compile, holding the cycle lock** so no cycle runs on half-patched files.

```bash
cd /opt/data/polybot && flock -w 900 data/.cycle.lock -c "cd /opt/data && patch -p1 --fuzz=0 < CO-006.diff && cd polybot && .venv/bin/python -m compileall -q polybot tools && echo APPLIED_AND_COMPILED"
```

The last line must be `APPLIED_AND_COMPILED`. If it is not: run the rollback in section 7, then STOP and report.

**Step 5. Record Chase's approval in the pre-registration, then register the scoring files.**

```bash
cd /opt/data/polybot && sed -i 's/(CO-006, proposed 2026-11-16; pending approval by Chase)/(CO-006; approved by Chase on 2026-11-16)/' notes/PREREGISTRATION.md && grep -c "(CO-006; approved by Chase on 2026-11-16)" notes/PREREGISTRATION.md
```

That must print `1`. Then:

```bash
cd /opt/data/polybot && .venv/bin/python tools/scorecard.py --register "CO-006 master (approved by Chase 2026-11-16): blend 18:00 gate, edge-sorted candidates, fee-inclusive caps, price ceiling 0.77, section 7 evaluation, settlement pays only on final 1/0 prices."
```

**Step 6. Run one cycle by hand.**

```bash
cd /opt/data/polybot && ./cycle.sh
```

**Step 7. Verify.**

```bash
cd /opt/data/polybot && .venv/bin/python -c "
from polybot import db
con = db.connect()
since = \"strftime('%Y-%m-%dT%H:%M:%SZ','now','-2 hours')\"
cols = con.execute(\"SELECT count(*) FROM pragma_table_info('blend_snaps')\").fetchone()[0]
snaps = [tuple(r) for r in con.execute(f'SELECT blend_version, count(*), sum(tradable) FROM blend_snaps WHERE ts >= {since} GROUP BY 1')]
orders = [tuple(r) for r in con.execute(f\"SELECT status, count(*) FROM orders WHERE strategy='blend' AND ts >= {since} GROUP BY 1\")]
waiting = con.execute('SELECT count(*) FROM settle_skips').fetchone()[0]
print('cols', cols); print('snaps', snaps); print('orders', orders); print('waiting', waiting)
ok = cols == 27 and any(v == '2026-11-16-v2' for v, _, _ in snaps)
print('RESULT:', 'OK' if ok else 'ROLL BACK')
"
```

```bash
cd /opt/data/polybot && .venv/bin/python tools/scorecard.py | grep -n "registration\|Blend pre-registered\|FLAG"
```

```bash
cd /opt/data/polybot && awk '/^===== cycle/{buf=""} {buf=buf $0 "\n"} END{printf "%s", buf}' "data/log/$(date -u +%F).log" | grep -n "Traceback\|blend failed\|scorecard FAILED"
```

All of these must hold:

- First command ends `RESULT: OK` (27 columns, and the new rows carry `2026-11-16-v2`).
- The scorecard shows "scoring files and thresholds match the registration", shows
  "Blend pre-registered evaluation (PREREGISTRATION §7) → **PENDING", and shows no line containing `FLAG`.
- The third command (it searches only the cycle you just ran) prints nothing.

If any of them fails: run the rollback in section 7, then STOP and report with the output. Do not try to repair the
patch on the server.

Not failures, just things to report: `snaps` showing zero tradable rows (no market had reached 18:00 station time on
its lead-2 day during that cycle), a few rejected orders, a small `waiting` count.

**Step 8. Log it and report to Chase.** Append to `notes/CHANGELOG.md`: the date, "CO-006 master applied (approved by
Chase 2026-11-16)", the patch sha256, the register note, and the outputs of steps 3 and 7. Then send Chase a short
report: which steps ran, the two `RESULT` lines, free disk, and anything listed under "things to report".

**The next day, once.** Run the first command of step 7 again and tell Chase the `waiting` and `orders` numbers. A
`waiting` count that has grown and holds markets more than two days past their date means the settlement rule is
holding real results: report it, do not change the code. Hundreds of rejected blend orders would mean the rejection
spam is back: report it.

## 7. Rollback

```bash
cd /opt/data && patch -R -p1 --fuzz=0 < CO-006.diff
```

Then re-register the manifest with a note saying so. The reverse patch restores the nine files exactly (tested).
It does not undo data: the new columns, the `settle_skips` table, the registration history, the epoch boundary and any
orders placed in between remain, and they are harmless to the older code. Never delete rows to make a rollback look clean.

## 8. Verification record

| Check | Result |
|---|---|
| `patch -p1 --fuzz=0` on snapshot + CO-005 | Clean, 9 files, 22 hunks |
| Hunk headers read for deletions | None. Pre-registration hunk is `@@ -94,3 +94,46 @@` (append only) |
| Compile | Pass |
| Patched tree identical to the tested tree | Yes, byte for byte |
| Reverse patch restores the base | Yes |
| `verify_master.py` on Rev B alone | 12 of the first 20 tests fail (written first, before the fixes) |
| `verify_master.py` on this order | 24 of 24 pass |
| `verify_astra.py` on Carbon Rev A, Rev B, this order | 12, 15, 18 of 19 pass |
| Astra's remaining failure | A6 only, deferred in section 3 |
| End-to-end paths exercised by those suites | `evaluate_event` with the real `PaperBroker`, `dashboard.build`, `scorecard.py --register`, `scorecard.py`, migration of a 16-column table, settlement |
| Independent review of the diff (fresh-context reviewer, no authoring history) | No critical or high findings. 2 medium, both in the settlement guard, fixed and tested: the first version accepted 0.995 / 0.005 as final, and split resolutions had no review flag. 4 low: 3 fixed (no-epoch scoring, missing-date guard, effective dates defined in the text), 1 documented below |
| Known limit, not fixed | S3's band reuses the scorecard's shared band function, whose zero-loss floor was calibrated for returns per dollar, not per share. The sample and date-clustered errors still apply, and the power table above was produced with that same function, so it reflects the rule as shipped |
| Not done on the laptop | A cycle on the live ledger; a dry run on Hermes's current code; Gamma's real responses. Section 6 does all three on the server (steps 1, 3, 6) and stops or rolls back if any fails |

## 9. Decisions

Decided by Chase on 2026-11-16: apply this order now.

Still open, and none of them blocks the apply:

1. Whether `blend` should keep running after the evaluation. The honest odds are 10 to 15% that it clears costs.
   Paper trading is free, and section 7 of the pre-registration gives a clean answer in about 60 market dates.
2. The 30-date check. This order registers the wording from the pre-registration text: retire at 30 dates only if
   blend does not beat the market's log-loss at all, or its signal returns are clearly negative. Carbon's audit
   described a harsher version, which would retire a strategy with zero true edge about half the time at 30 dates but
   would also retire a real +4c edge about one time in eight. Changing it later needs a new order and a new epoch.

## 10. My mistakes that this order corrects

- Deployed a 15:00 gate and cited an 18:00 test for it.
- Ran the supporting replay with zero fees when the fee column said 5%.
- Let bucket order decide which trade fills the event cap.
- Left a price band that could never fill and wrote a rejection every scan.
- Wrote cap and cash checks that ignored the fee.
- Wrote a stopping rule without checking its power.
- Chose the baseline after seeing which was better.
- Never recorded that blend's scoring depended on a retired strategy's table.
- Wrote a docstring claiming live forecasts are never staler than the training archive. They are.
