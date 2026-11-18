# The audits

On 2026-11-16 the `blend` change order (CO-005) was given to three independent AI reviewers, named here Argon, Carbon
and Astra. Each audited the code, the model and the evidence. The author of CO-005 (Claude) then had to answer every
finding with executable evidence, and one reviewer answered the author's objections in turn.

```
 CO-005 ──▶ Argon audit ──▶ Carbon audit ──▶ Astra audit
                 │               │               │
                 └───────────────┴───────────────┘
                                 ▼
                 author cross-examination (evidence only)
                                 ▼
                 Carbon responds: concedes 13 points, reissues its patch
                                 ▼
                 master CO-006: confirmed findings only
```

The rule for cross-examination: challenge every claim, do not fight what the data shows, and dismiss nothing without
proof. Procedure: [AUDIT_SOP.md](../audits/AUDIT_SOP.md). Full verdicts: [AUTHOR_REBUTTAL](../audits/AUTHOR_REBUTTAL_2026-11-16.md).

## Argon

Files: [audit](../audits/argon/AUDIT_CO-005_blend_2026-11-16.md), [proposed order](../audits/argon/CO-006-blend-evaluation-and-logging.md).

| Finding | Verdict |
|---|---|
| The 20-date stopping rule cannot detect a working strategy | Right in direction; its power figures were 2 to 3 times too pessimistic |
| The new model's gain over the old one was not shown | Mostly right |
| The "unseen" dates had been used to choose the rule | Right |
| A claim that live inputs are never older than training inputs | Right: the claim was false, effect small |
| Buying at the ask is where the bot loses | Right |
| Data usage was several times the estimate | Right on usage, wrong on the plan |
| The log could not reconstruct decisions | Right |
| The baseline covered the wrong markets | Right |

## Carbon

Files: [audit](../audits/carbon/AUDIT_CARBON_CO-005_2026-11-16.md), [proposed order](../audits/carbon/CO-006-carbon-execution-scoring-and-governance.md).

| Finding | Verdict |
|---|---|
| Fees were left out of the replay | Confirmed. Fees applied on every date since 2026-11-03 |
| The blend's advantage on Brier score is about zero | Confirmed. The advantage is in log-loss only |
| The bot started at 15:00 but the evidence was for 18:00 onward | Confirmed |
| Bucket order crowded the event limit | Confirmed. Sorting by edge is most of the measured gain |
| Moving the start to 18:00 improves returns | Not supported: +$5 (−49 to +52) |
| A price ceiling improves returns | Wrong: it changes replay profit by exactly $0 |
| A claimed +$82.79 improvement | Overstated: +$43 (−9 to +90) on a paired replay |
| Several function and setting names | Did not exist in the code |
| Its patch | Deleted six sections of the pre-registration; must not be applied as written |

Carbon conceded all 13 contested points and issued a corrected patch, which became the base of the master order.

## Astra

Files: [audit](../audits/astra/AUDIT_ASTRA_CO-006_2026-11-16.md), [amendment requests](../audits/astra/CO-006-ASTRA-AMENDMENT-REQUESTS.md).

| ID | Finding | Verdict |
|---|---|---|
| A1 | The proposed patch deletes the pre-registration | Confirmed, blocker |
| A2 | The go-live check ignores the new pass rule | Confirmed, fixed |
| A3 | Limit and cash checks leave out the fee | Confirmed, severity overstated: largest overshoot under 10 cents |
| A4 | Settlement can pay on a closed but non-final market | Code path confirmed; never occurred in 2,224 resolved events; fixed |
| A5 | Settlement sweep ignores the new strategy's log | Confirmed latent, fixed |
| A6 | Maker-fill rule ignores trade direction | Contested, affects a retired path, deferred on purpose |
| A7 | A future timestamp is accepted as fresh | Confirmed, fixed |
| A8 | A logging change alters the scoring period | Confirmed |
| A9 | A partial fill is recorded as filled | Confirmed, cosmetic |

## The author's own mistakes

Among those recorded in the rebuttal and in CO-006:

1. Wrote a stopping rule without checking its power.
2. Treated a better forecast score as a trading edge without checking that the retired strategy had the same score.
3. Chose the rule on dates described as unseen.
4. Made a false freshness claim about live inputs.
5. Understated data usage about seven times.
6. Left fees out of the replay.

## What the audits did not change

No audit found an edge that had been missed. After every correction the best rule replays at +2.4% with an interval
from −5.6% to +10.9%. See [RESULTS.md](RESULTS.md).
