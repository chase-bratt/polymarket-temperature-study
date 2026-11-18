# Polybot: Astra Amendment Requests for CO-006

Status: REQUESTED, NOT IMPLEMENTED OR APPROVED. Date: 2026-11-16. These requests amend the reviewed Carbon proposal or define a tightly scoped follow-up; they do not allocate a conflicting CO number, supersede Carbon's quantitative work, or authorize production changes.

Target Carbon diff SHA-256: `9981e95da300a17da70da45c3a8ae4ae5e5441e4f68d4d003a0e142b432b2255`. Evidence and confidence for each ID are in `AUDIT_ASTRA_CO-006_2026-11-16.md` and its executable test results.

## Required before sign-off

- **A1, preserve governance:** Amend only the intended preregistration section or append section 7 while retaining sections 1–6 and clear supersession wording. Do not assert owner approval before it exists. Acceptance: original standing sections remain, every internal section reference resolves, and the human reviews the exact preregistration diff before registration.
- **A2, unify readiness:** Route blend promotion through the new final evaluator and integrity verification, then the existing operational checklist. An absent manifest, missing evaluation input, pending/futility result, or integrity mismatch must make readiness false. Acceptance: the 20-date counterexample fails readiness; a separately constructed complete final-pass fixture can pass; corrupting a locked file forces false. Do not weaken any numeric threshold merely to turn tests green.
- **A3, enforce all-in constraints:** Introduce one deterministic fill/spend calculation shared by strategy preflight and broker validation, including fee rounding and available depth. Reserve at most the minimum of available cash, remaining event budget, and remaining open budget; retain venue minimum quantity after any resize. Acceptance: 0.78/minimum-five-share boundary, near-zero cash, accumulated event exposure, and multilevel/partial-fill cases cannot overspend or leave negative cash. No order/fill/position/cash mutation on rejection.
- **A4, fail closed at settlement:** Separate market closure from final payout authority and validate the final payout vector for the supported market type. Unknown, disputed, malformed, or unsupported responses must leave positions unsettled with an auditable reason. Acceptance: the `[0.6,0.4]` missing-status fixture does not pay; valid final YES and NO fixtures pay correctly exactly once; fetch/history-capture failures cannot commit half a settlement.

A1 and A2 belong in a corrected CO-006. A3 and A4 touch files Carbon excluded; either explicitly expand and re-review its scope or assign a separate owner-approved execution/settlement change order. Do not conceal those edits inside a “documentation-only” amendment.

## Follow-up requirements without duplicating Carbon

- **A5, independent outcome universe:** In the full settlement sweep, include unresolved markets in `blend_snaps` as well as legacy snapshots and held positions. Preserve the positions-only fast path. Acceptance: unheld blend-only buckets receive verified outcomes, and missing outcomes are reported rather than silently dropped.
- **A6, direction-aware maker matching:** Specify the API's aggressor-side semantics using captured examples, then filter own-outcome SELL and appropriately mapped complementary trades. Acceptance: own-outcome BUY prints cannot fill a resting bid; valid SELL prints, complementary cases, duplicate prints, and queue depletion have explicit fixtures. Keep retired strategies retired.
- **A7, as-of guard:** Reject or quarantine cache timestamps after the decision time; do not repair historical replay by fetching present-day data. Acceptance: negative ages cannot be tradable, strict replay never uses a later publication/fetch timestamp, and refresh failure remains observable.
- **A8, quota/epoch runbook:** State that refresh cadence changes can alter both feature freshness and the epoch hash despite an unchanged version. Acceptance: the documentation matches the current hashing behavior, the change is authorized, and evaluation history is not pooled across the new epoch by silently excluding the field.
- **A9, partial status contract:** Document whether `filled` means terminal or fully satisfied. If the latter, record `partial` for insufficient depth and verify downstream acceptance of both states; preserve actual fills and cash.

## Integration, apply, and rollback requirements

The implementer must provide a new hash-identified diff and rerun forward zero-fuzz application, compilation, all adversarial tests, scorecard registration/rendering, dashboard rendering, and reverse-patch comparison in a disposable copy. New tests must cover the complete final-pass path and final-resolution idempotency, not only the failing counterexamples.

Before an approved deployment, capture a consistent SQLite backup, runtime hashes, current config, scoring manifest, and pending order state. The production runbook must stop relevant writers, apply only the approved diff, register the approved locked-file changes with the actual approval reason, run one controlled paper cycle, and verify cash/fill/position/outcome invariants.

Rollback must distinguish source restoration from data restoration: additive SQLite columns may remain compatible, but registrations, epoch history, settlements, and intervening orders cannot be undone by `patch -R`. Never discard post-backup trades or overwrite a live ledger merely to make a rollback appear clean.
