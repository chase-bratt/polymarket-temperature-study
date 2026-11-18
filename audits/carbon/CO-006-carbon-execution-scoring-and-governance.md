# CO-006 (Carbon Superseding, Rev B): `blend` Execution Alignment, Fee-Inclusive Cap Guards, Candidate Ordering, and Pre-Registered `scorecard.py` Implementation

- **Change Order ID:** `CO-006` (`Carbon Superseding Edition, Rev B — Post-Fable 5.2 Adversarial Code Audit`)
- **Author / Auditor:** Gemini 4 Carbon (Lead Independent Quantitative, Data & Pipeline Auditor), incorporating all verified patch and citation fixes from Claude Fable 5.2's `2026-11-16` clean-room review
- **Date:** 2026-11-16
- **Status:** `PROPOSED (PENDING APPROVAL BY CHASE)` — Supersedes unimplemented `change_orders/2026-11-16/Argon/CO-006-blend-evaluation-and-logging.md` and `Argon/CO-006.diff`, and replaces Carbon `Rev A`. Requires Chase's explicit approval before applying on Hermes because it updates live `blend` execution parameters (`blend_version = "2026-11-16-v2"`), `polybot/scoring.py`, `tools/scorecard.py`, and `notes/PREREGISTRATION.md`.
- **Priority:** `P0` (Live execution timing/crowding-out bugs, fee-inclusive cap/cash boundary, `orders` rejection spam, and unimplemented `scorecard.py` §7 baseline/evaluation/readiness gate).
- **Confidence:** **High** that every patched bug exists in post-`CO-005` code (`snapshots/polybot_code_2026-11-14` + `CO-005.diff`) and that `CO-006.diff` (`Rev B`) applies cleanly (`patch -p1 --fuzz=0`, `py_compile`, tests `A–F`, plus all 4 of Fable 5.2's reproducer tests verified); **Low-to-Medium** that `blend` has positive true out-of-sample EV after costs (`+2.5%`, `95%` band `-7.3%` to `+13.6%` historical fees; `+4.3%`, `-6.5%` to `+16.2%` under `0.05` fees across `23` dates), which is why `blend` remains **strictly paper-only** (`"mode": "paper"`).

---

## Rev B Corrections Integrated from Claude Fable 5.2 Adversarial Review (`2026-11-16`)

1. **Preserves `notes/PREREGISTRATION.md` §§1–6 in full (`@@ -94,3 +94,21 @@`, `96 -> 114` lines):** Fixes `Rev A`'s accidental truncation hunk (`@@ -1,96 +1,19 @@`).
2. **Removes premature approval language:** `PREREGISTRATION.md` §7 is marked `proposed 2026-11-16; pending approval by Chase`; zero added lines in `CO-006.diff` assert `"approved by Chase"`.
3. **Wires `PREREGISTRATION` §7 60-date gate into `scoring.go_live()`:** For `strategy == "blend"`, `scoring.go_live(con, cfg, "blend", ...)` appends `("PREREGISTRATION §7 60-date evaluation PASS", bev["verdict"], bev["verdict"].startswith("PASS"))`, ensuring `ready` stays `False` at 20 dates / 100 trades while §7 is `PENDING`.
4. **Fee-inclusive price ceiling (`0.77`) and cap/cash checks (`blend.py`):** Under `min_size = 5` and `5%` fee rate, `ask = 0.78` fills `5.1` shares for `$3.978` plus `$0.04376` fee = `$4.02176` (`> $4.00` `blend_max_event_usd`) and could drive `$3.978` cash to `-$0.04376`. `Rev B` lowers `blend_max_price` to `0.77` (`$3.927 + $0.04516 = $3.97216 <= $4.00`) and checks `est_total = round(usd * (1 + fee_rate * (1 - ask)), 5)` against `blend_max_event_usd`, `blend_max_open_usd`, and `db.cash_balance(con)`.
5. **Drops `dashboard.py` hunk & corrects prose citations:** Removes the `dashboard.py` hunk (which inverted fresh `Mark` vs stale `Market now` on Asian settlement-day rows and risked colliding with Hermes's post-snapshot UI edits) and fixes all prose references to real code identifiers (`"mode": "paper"`, `blend_max_event_usd`, `scoring.current_epoch_start`, `tempmodel.PEAK_HOUR = 15`, 5M/month Open-Meteo Professional quota, and `governance.strategy_sha` sensitivity to `blend_refresh_min`).

---

## Why This Change Order Supersedes Argon `CO-006`

1. **3-Hour Timing Mismatch (`15:00` vs `18:00` local):** `pipeline/research_edge_after_costs.py` (2026-11-14, line 62) coded `late` as `hours_to_day_start <= 24 * (s.lead - 1) + 6` (`<= 30h`, i.e. **`>= 18:00` station local time** at `lead = 2`), whereas `CO-005` gated `blend` implicitly at **`15:00` local** via `m_lead == lead` and `tempmodel.PEAK_HOUR = 15` (`polybot/tempmodel.py` line 38; `polybot/blend.py` line 132). Adding explicit `blend_min_local_hour = 18.0` aligns live trading with the leak-free `late` window.
2. **Ascending-Bucket Crowding Out:** `blend.evaluate_event()` (`CO-005.diff`, 2026-11-14) iterated `ev["markets"]` in ascending temperature order (`lo` low to high). Because `min_size = 5` shares at `NO = 0.72..0.77` scales a `$2.00` stake up to `$3.67..$3.93` (`5 * price * 1.02`), a marginal `0.081` edge on a low shoulder bucket exhausted the `$4.00` `blend_max_event_usd` cap before a `0.18+` edge on the modal bucket was reached.
3. **Unreachable Dead Price Zone (`ask >= 0.78`), Fee Cap Overflow, & `orders` Rejection Spam:** Once `5%` taker fees are included, any `ask >= 0.78` requires `5 * 1.02 * ask * (1 + 0.05 * (1 - ask)) >= $4.02176 > $4.00` (`blend_max_event_usd`). Previously, `0.79..0.90` generated `128` unreachable `[tiny]` rejections in replay, `0.78` overflowed `blend_max_event_usd` by `$0.02176` after fees, and `blend.py` logged `broker.reject()` rows for `[held]`, `[opposite held]`, and `[event cap]` every 30 minutes (`1,194+` duplicate rejections in replay).
4. **Missing Depth & Local-Hour Columns + Non-Deterministic `inputs()` Clock:** Argon's 8 `blend_snaps` columns omitted `local_hour`, `yes_ask_depth`, and `no_ask_depth`, and `blend.inputs()` called `dt.datetime.now()` directly instead of `now_utc`.
5. **Unimplemented `PREREGISTRATION` §7 Scorer, Readiness Gate, & `scorecard.py` Baseline:** Argon amended `PREREGISTRATION.md` §7 in prose without updating `scoring.py` or `tools/scorecard.py`. Meanwhile, `scorecard.py` benchmarked `blend` against `max(B1, B2)` chosen ex-post from `scoring.baselines_forecast()` (`e.kind = 'highest' AND s.lead_days IN (1, 2)`, first scan of the event including Hong Kong and Taipei, with `B2` paying the zero-fee maker `NO` mid rather than `blend`'s taker `ask + fee`).

---

## Patch Scope (8 Files Modified Against Post-`CO-005` Codebase)

| File | Exact Changes |
|---|---|
| `polybot/config.json` | Sets `"blend_min_local_hour": 18.0`, `"blend_max_price": 0.77`, `"blend_version": "2026-11-16-v2"` (retains `"mode": "paper"`). |
| `polybot/polybot/config.py` | Updates `Config` dataclass defaults (`blend_min_local_hour: float = 18.0`, `blend_max_price: float = 0.77`, `blend_version: str = "2026-11-16-v2"`). |
| `polybot/polybot/blend.py` | (1) `SNAP_EXTRA` adds **11 columns** idempotently (`input_age_min`, `local_hour`, `hours_to_peak`, `tradable`, `edge_yes`, `edge_no`, `fee_rate`, `yes_ask_depth`, `no_ask_depth`, `model_name`, `blend_version`); (2) `inputs(..., now=now_utc)` uses `now_utc` deterministically and records `cache["_refresh_fail"]`; (3) `evaluate_event()` sorts eligible candidates by descending `(edge, p_side)`, includes taker fees (`est_total`) in `blend_max_event_usd`, `blend_max_open_usd`, and `db.cash_balance()` checks, and skips `[held]`, `[opposite held]`, and `[event cap]` before calling `broker.reject()`. |
| `polybot/polybot/__main__.py` | Prints a concise warning line per scan listing any stations in `bl_cache["_refresh_fail"]`. |
| `polybot/polybot/tempmodel.py` | Clarifies `model_lead()` docstring regarding live `K=0` publication delay vs historical `_previous_day2`. |
| `polybot/polybot/scoring.py` | (1) Decouples `fill_realism(..., strategy='blend')` onto `blend_snaps`; (2) adds `baselines_blend()` (`B1_blend` and `B2_blend` paying taker `ask + fee` on `blend`'s exact tradable universe); (3) adds deterministic `blend_evaluation()` (`S1_logloss`, `S1_brier`, `S2_vs_forecast`, `S3_per_share` with 30-date futility and 60-date pass gates); (4) wires `blend_evaluation()` `PASS` check into `scoring.go_live()` for `blend`. |
| `polybot/tools/scorecard.py` | Benchmarks `blend` go-live row 5 against single pre-registered `B2_blend` (taker `NO` ask+fee) and renders the `Blend pre-registered evaluation (PREREGISTRATION §7)` section. |
| `polybot/notes/PREREGISTRATION.md` | Preserves all 96 lines of Title + §§1–6 intact and appends §7 (`proposed 2026-11-16; pending approval by Chase`). |

**Explicitly Excluded from `CO-006`:** `models/temperature_v1/model.json` weights, `dashboard.py`, `broker.py`, `execution.py`, `clob.py`, `settle.py`, `governance.py`, and real-money execution (`"mode": "paper"` unchanged; `forecast` and `follow` remain retired via SQLite `strategy_status`).

---

## Impact and Epoch Analysis

1. **Historical 23-Date Replay Bracket (`data/ml/market/blend_bucket_scan.parquet`, `290,950` rows, `models/temperature_v1/model.json`):**
   - Because Open-Meteo's archive stores 24h `previous_dayK` steps while live `K=0` at `15:00..18:00` local is `~54h–61h` before target peak (between `fc2` at `48h` and `fc3` at `72h`), replay P/L is bounded by two archive brackets:
     - **Pessimistic archive bound (`knowable`: `fc3` for `15:00..18:00`, `fc2` for `>=18:00`):** shifts from `-$50.09` (`-2.9%`, `n=601`) to `+$16.93` (`+1.3%`) from the `18:00` gate (`+$67.02`), and to `+$32.70` (`+2.5%`, `-7.3%` to `+13.6%`, `n=454`) from descending-edge sorting (`+$15.77`), totaling `+$82.79`.
     - **Optimistic archive bound (`optimistic`: `fc2` throughout `>=15:00`):** starts at `+$12.15` (`+0.8%`, `-7.5%` to `+9.7%`, `n=519`), shifts `+$4.78` to `+$16.93` from the `18:00` gate, and `+$15.77` to `+$32.70` from descending-edge sorting (`+$20.55` total; `+$36.20 -> +$50.34` under `0.05` fees).
   - **Zero P/L credit claimed for `blend_max_price`:** On `0.01`-tick books, asks in `[0.79, 0.90]` could never fill previously (`$0.00` P/L impact; removing them eliminates `[tiny]` rejection spam), while lowering `0.78 -> 0.77` prevents `$0.02176` post-fee `$4.00`-cap overflows at `ask = 0.78`.
2. **Clean Epoch Boundary (`blend_version = "2026-11-16-v2"`):**
   - Bumping `blend_version` to `"2026-11-16-v2"` causes `scoring.current_epoch_start(con, cfg, "blend")` to start a clean epoch on the first scan after deployment while existing `2026-11-14-v1` positions settle normally via `settle.py`.

---

## Pre-Deployment Test Evidence (`Verified 2026-11-16, Rev B`)

Tested in `/tmp/polybot_verify/fable_repro_test` against `snapshots/polybot_code_2026-11-14` + `CO-005.diff`:
- **Patch Cleanliness:** `patch -p1 --fuzz=0 < CO-006.diff` -> `exit 0` across all 8 files (`0` fuzz, `0` offsets).
- **Fable Reproducer #1 (`PREREGISTRATION.md`):** Preserves all `96` existing lines (Title + §§1–6) and appends §7 (`114` lines total).
- **Fable Reproducer #2 (Approval Hygiene):** Zero added lines in `CO-006.diff` contain `"approved by Chase"`.
- **Fable Reproducer #3 (`go_live` §7 Gate):** Fixture of `100` winning settled trades across `20` effective dates with confirmed capacity passes all `8` legacy §2 checks while `blend_evaluation()` returns `"PENDING (0.0/30.0 effective dates)"` and `scoring.go_live(con, cfg, "blend", 0.0)["ready"]` returns **`False`**.
- **Fable Reproducer #4 (Fee-Inclusive `0.77` Ceiling & Cash/Cap Guards):** At `ask = 0.78` (`5.1` shares -> `$4.02176` with fee), `0` fills occur (both under `blend_max_price = 0.77` and even if overridden to `0.90`); at `ask = 0.77` with `$3.95` cash (`< $3.97216`), rejects cleanly leaving cash at `+$3.95`; with `$3.978` cash, fills `5.1` shares with `event_exposure = $3.97216 <= $4.00` and remaining cash `+$0.00584 >= $0.00`.

---

## Apply Runbook (Hermes `/opt/data/polybot` — Execute ONLY After Chase Approves)

```bash
# 1. Pause polybot crons; confirm no python cycle is mid-run.
# 2. Backup SQLite ledger before patching:
cp /opt/data/polybot/data/polybot.sqlite /opt/data/polybot/data/polybot.sqlite.bak-pre-co006

# 3. Dry-run with --fuzz=0 and apply from /opt/data (parent of polybot/):
cd /opt/data
patch -p1 --fuzz=0 --dry-run < /path/to/Carbon/CO-006.diff
patch -p1 --fuzz=0 < /path/to/Carbon/CO-006.diff

# 4. After Chase's explicit approval, update §7 heading in polybot/notes/PREREGISTRATION.md from
#    'pending approval by Chase' to 'approved by Chase on <YYYY-MM-DD>', then compile & migrate:
cd /opt/data/polybot
.venv/bin/python -m py_compile polybot/*.py tools/scorecard.py
.venv/bin/python -c "from polybot import db, blend; c=db.connect(); blend.ensure(c); print(len(c.execute('PRAGMA table_info(blend_snaps)').fetchall()))"

# 5. Run one manual cycle to log config_log epoch '2026-11-16-v2':
./cycle.sh

# 6. Re-register SCORING_MANIFEST.json (ONLY after Chase approves):
.venv/bin/python tools/scorecard.py --register "CO-006 Rev B: blend_min_local_hour=18.0, blend_max_price=0.77 fee-inclusive cap, descending-edge sort, 11 blend_snaps cols, B2_blend + S1a/S1b/S2/S3 + go_live gate, blend_version=2026-11-16-v2"
.venv/bin/python tools/scorecard.py | head -n 95
```

---

## Quota & Epoch Note (`Open-Meteo 5M/Month Professional Plan`)

- At `blend_refresh_min = 55.0`, `blend` uses roughly `~700k` weighted Open-Meteo units/month (`~14%` of Chase's **5M/month Professional plan**, verified in `1_analysis_and_ML_guidance_2026-10-29_to_2026-11-13.md`), so no refresh change is needed.
- **Important governance note:** Because `governance.strategy_keys("blend", cfg)` hashes every config key starting with `"blend_"`, changing `"blend_refresh_min"` **does** alter `governance.strategy_sha("blend", cfg)` (`08a090ebde` -> `2aad33ac89` under `Rev A`, and correspondingly under `Rev B`), which starts a new `blend` epoch and counts against `freeze_days = 7`. Do not change `blend_refresh_min` mid-epoch unless prepared to reset the epoch clock.
