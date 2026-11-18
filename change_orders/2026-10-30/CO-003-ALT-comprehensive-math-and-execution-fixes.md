# CO-003: fix `forecast` overconfidence, passive-bid adverse selection, `follow`/`intraday` math bugs, and cluster governance

- **Status:** ready to apply (`CO-003A` entry pause is risk-reducing and allowed immediately under the 7-day freeze; `CO-003B–F` bump `forecast_version = "2026-10-30-co003-calibrated"` to start a clean scoring epoch)
- **Risk:** reduces risk (`CO-003A` halts negative-EV passive fills while keeping `snapshots` and resting-bid shadow evaluation running; `CO-003B–F` fix systematic probability overconfidence, adverse selection on passive bids, same-event correlated `follow` double-losses, left-truncation distortion in `intraday`, and small-cluster SE underestimation in `governance`)
- **Patch:** `CO-003.diff`
- **Files:**
  - `polybot/polybot/config.py`
  - `polybot/polybot/probability.py`
  - `polybot/polybot/strategy.py`
  - `polybot/polybot/follow.py`
  - `polybot/polybot/db.py`
  - `polybot/polybot/intraday.py`
  - `polybot/polybot/fit.py`
  - `polybot/polybot/governance.py`

---

## Why

Across the two export periods (Oct 21–25 in `polybot_performance_2026-10-25.xlsx` and Oct 25–30 in `ANALYSIS_forecast_2026-10-30.md`), six concrete mathematical and microstructure defects were verified in the live paper ledger and codebase:

1. **`forecast` Probability Overconfidence (+12.1% to +18.3% gap) from Proportional De-Vigging + Modal Concavity + Optimizer's Curse:**
   - Across all `190` settled `forecast` trades in epochs `d5e06839b5` (`137` taker trades) and `f312e9d5ff` (`53` passive maker trades), actual wins were **`111` (`58.42%`)** against **`109.2` (`57.47%`)** market-implied wins (two-sided binomial $p = 0.826$) and **`132.9` (`69.95%`)** pool-implied wins ($z = -3.47$, $p = 0.00084$).
   - Isolating the **`107` post-Oct-25 settled trades** in `d5e06839b5` (subtracting the Oct 25 snapshot of `30` settled, `21` won, `16.87` market-implied, `20.00` pool-implied, `+$20.84` P&L from the Oct 30 total of `137` settled, `83` won, `79.4` market-implied, `95.2` pool-implied, `+$16.90` P&L) shows `62/107` wins (`57.94%`) vs `62.53` market-implied (`58.44%`, error `-0.50%`) and `75.20` pool-implied (`70.28%`, overstatement **`+12.34%`**), losing **`-$3.94`** (`-1.21%` ROI).
   - On the Oct 29 daily trade-free review of `193` live-rule favourites, `NO` won `109/193` (`56.48%`) vs `108.3/193` (`56.11%`, $p = 0.942$) market-implied and `131.4/193` (`68.08%`, $z = -3.46, p = 0.00085$) pool-implied.
   - **Three compounding root causes in `probability.py` and `strategy.py`:**
     1. *Proportional ask-heavy normalization:* `p_mkt = probability.normalise(mids)` substitutes `ask` (`1¢–2¢`) when `bid is None` on tail buckets, inflating $\sum_{j=1}^K \text{mid}_j \approx 1.10\text{–}1.15$ and dividing every bucket by $\sum \text{mid}_j$, which depresses the modal favourite `YES` probability by `3–5¢` before the model is even consulted ([Shin 1993](https://academic.oup.com/ej/article-abstract/103/420/1141/5157258); [Clarke, Kovalchik, & Ingram 2017](https://www.sciencepublishinggroup.com/article/10.11648/j.ajss.20170506.12)).
     2. *Modal Concavity (Jensen Attenuation) of the Logarithmic Pool:* Combining two bell-shaped distributions with location difference $\varepsilon = \mu_{\text{mod}} - \mu_{\text{mkt}}$ via $p_j^{\text{pool}} \propto (p_j^{\text{mod}})^w (p_j^{\text{mkt}})^{1-w}$ evaluates at the market mode $x^* = \mu_{\text{mkt}}$ to $p_{j^*}^{\text{pool}}(\varepsilon) \approx p_{j^*}^{\text{mkt}} \exp(-w^2 \varepsilon^2 / 2\sigma^2) < p_{j^*}^{\text{mkt}}$ for **every** $\varepsilon \ne 0$ ([Ranjan & Gneiting 2013](https://projecteuclid.org/journals/electronic-journal-of-statistics/volume-7/issue-none/Combining-predictive-distributions/10.1214/13-EJS823.full)). Even a zero-mean NWP error ($\mathbb{E}[\varepsilon]=0, \text{Var}(\varepsilon)=\tau^2$) strictly attenuates the modal `YES` probability by $\exp(-w^2\tau^2 / 2\sigma^2)$ and inflates `NO` by `9–12%`.
     3. *Optimizer's Curse (Post-Selection Bias):* Conditioning on `edge >= 0.08` selects the upper tail of estimation noise without Empirical-Bayes shrinkage ([Smith & Winkler 2006](https://pubsonline.informs.org/doi/10.1287/mnsc.1050.0451)), even on `highest` at `lead_days <= 1` where the market NLL (`1.2256`) beats both raw model (`1.3910`) and log pool (`1.2860`).

2. **Adverse Selection on Passive Resting Bids in `f312e9d5ff` (`-$13.67`, `-10.0%` ROI):**
   - In epoch `f312e9d5ff` (`123` resolved bids: `53` filled, `70` unfilled), **unfilled bids won `55/70` (`78.57%` vs `43.7` implied, `+25%` counterfactual ROI)**, whereas **filled bids won `28/53` (`52.83%` vs `30.5` bid-implied, `-14%` ROI)**.
   - The fill probability on eventual losers was $25/40 = 62.50\%$ vs $28/83 = 33.73\%$ on eventual winners — an **Adverse Fill Ratio** of $\hat{\lambda} = 1.853$ (Fisher's exact $p = 0.00347$; date-stratified Cochran-Mantel-Haenszel $p = 0.00245$).
   - By the [Glosten & Milgrom (1985)](https://linkinghub.elsevier.com/retrieve/pii/0304405X85900443) execution covariance identity, $\text{Cov}(\mathbf{1}_{\text{Fill}}, Y - b) = -0.0566$ (`-5.66¢` per bid placed, or `-13.14¢` per filled trade), concentrated on Oct 28 (`12/30` wins, `-$19.76` P&L) when dry offshore winds in Beijing (`21°C` actual vs `16.9°C` forecast) and Busan (`21°C` actual vs `18.6°C` forecast) warmed afternoon METARs into the favourite bucket and informed takers swept `polybot`'s stale resting `NO` bids (`56 of 108` fills were triggered by complement `YES` buys).

3. **`CO-001` Residual Defects in `follow.py` (`backtest` Crash + Correlated Same-Event Double-Losses):**
   - In `CO-001.diff`, `follow.backtest` (line 253) left `obs_only` inside `if ok:` *after* requiring `stt is not None` (`future_extreme`), and omitted `exact=(ev.get("rule") == "metar")`, causing `backtest` to still depend on `future_extreme` even when `follow_obs_only = True`.
   - Furthermore, on 2026-10-25, `follow` lost `-$9.98` in Kuala Lumpur (`31°C YES + 32°C NO`) and `-$10.00` in Wuhan (`26°C YES + 27°C NO`) by buying `YES` on bucket $B_1$ and `NO` on adjacent bucket $B_2 = B_1 + 1^\circ\text{C}$ in the same scan — two legs with $100\%$ loss correlation whenever temperature rises $1^\circ\text{C}$ into $B_2$. `follow` also overlapped with `forecast` on 10 markets (`20` positions).

4. **Left-Truncation vs. Left-Censoring Bug in `intraday.py` and Alternating-Index CV Leakage in `fit.py`:**
   - `intraday.bucket_probs` computed `(cdf(b) - cdf(a)) / (1 - cdf(floor))`, which is a **left-truncated** Gaussian $\mathbb{P}(X_{\text{rem}} \in B_j \mid X_{\text{rem}} > M_t)$ rather than the **left-censored** Gaussian law of $M_{24} = \max(M_t, X_{\text{rem}})$ ([Tobin 1958](https://doi.org/10.2307/1907382)), smearing the point mass $\Phi((\text{floor} - \mu)/\sigma)$ across impossible upper tail buckets whenever afternoon temperatures cool below $M_t$.
   - `fit.py` (`cases[0::2], cases[1::2]`) and `intraday.py` (`r[k::2]`) split train/test folds by alternating indices rather than chronological dates, leaking same-date regional weather shocks across train and test sets.

5. **`CO-002` Residual Defect in `governance.py` (`_cluster_se` Smaller Than Unclustered SE in All 7 Epochs):**
   - In `CO-002.diff`, `_cluster_se` computed `math.sqrt(G / (G - 1) * s2) / n` with equal trade weights and no small-cluster leverage or Student-$t$ correction. Because $G \in \{2, 3, 4\}$ and $24/30$ trades fell on one date, `_cluster_se` was **strictly smaller** than `max(se_sample, sd_floor/sqrt(n))` in all 7 historical epochs (`±7.0%` vs `±32.2%` on `d5e06839b5`; `±3.4%` vs `±12.9%` on `follow 1eb90fd31e`), making `max(se, _cluster_se)` a no-op ([Imbens & Kolesar 2016](https://www.gsb.stanford.edu/faculty-research/publications/robust-standard-errors-small-samples-some-practical-advice); [Carter, Schnepel, & Steigerwald 2017](https://ideas.repec.org/a/tpr/restat/v99y2017i4p698-709.html)).

---

## What changes

### 1. `CO-003A` — Freeze-Safe Entry Gate & Calibration Parameters (`polybot/polybot/config.py`)
- Adds `forecast_pause_entries: bool = True` (default `True` during the 7-day freeze per Option 2 of `ANALYSIS_forecast_2026-10-30.md`): `evaluate_event` continues writing every bucket's `forecasts` and `snapshots` rows for trade-free NLL/calibration scoring while pausing paper order execution on `forecast`.
- Adds calibrated parameters for when `forecast_pause_entries` is flipped to `False`:
  - `forecast_adverse_fill_ratio: float = 1.85` (empirical $\hat{\lambda} = 1.853$ from `f312e9d5ff`)
  - `forecast_eb_kappa_high_lead1: float = 0.15`, `forecast_eb_kappa_high_lead2: float = 0.55`, `forecast_eb_kappa_low: float = 0.65`
  - `forecast_passive_ttl_min: float = 55.0`, `forecast_max_yes_imbalance: float = 0.72`, `forecast_intraday_veto_c: float = 0.80`
  - `min_effective_dates: float = 15.0`, `moulton_rho_date: float = 0.15`
  - Bumps `forecast_version = "2026-10-30-co003-calibrated"`.

### 2. `CO-003B` — Spread-Weighted Power De-Vigging, EMOS Kernel, and Modal-Debiased Log Pool (`polybot/polybot/probability.py`)
- **`devig_market_probs(yes_books, no_books=None)`:**
  - Uses complement `NO`-book parity ($\text{yb}_j = \max(\text{yb}_j, 1 - \text{na}_j)$) when direct `YES` bids are absent.
  - Down-weights wide-spread two-sided quotes by $w_j = (1 - \text{spread}_j)^2$ toward the bid and quadratically dampens ask-only tail buckets ($0.25 \cdot \text{ya}_j^2$), then solves $\sum_{j=1}^K q_j^\kappa = 1$ for $\kappa \ge 1$ via bisection.
  - On a typical 9-bucket weather book where the modal favourite is `0.42/0.46` (`mid = 0.44`) and 6 tail buckets have `0.02` asks with no bid, old `normalise(mids)` depressed the favourite from `0.4400` to `0.4151` (`-2.5¢`); `devig_market_probs` restores it to `0.4595`.
- **`bucket_probs_emos(forecast_rec, buckets, unit, ...)`:**
  - Implements Ensemble Model Output Statistics ([Gneiting et al. 2005](http://journals.ametsoc.org/doi/10.1175/MWR2904.1)) with Silverman best-member kernel bandwidth $h = M_{\text{eff}}^{-1/5}$ ([Roulston & Smith 2003](http://journals.ametsoc.org/doi/10.1175/1520-0493(2003)131%3C1957:CEBMSA%3E2.0.CO;2)):
    $$\sigma_{\text{total}}^2 = c_0^2 + d_{\text{ens}} s_{\text{ens}}^2 + d_{\text{det}} s_{\text{det}}^2$$
    automatically widening the predictive distribution when the 4 deterministic runs (`GFS`, `ECMWF`, `ICON`, `GEM`) disagree (`sd ~ 1.18°C` vs `0.82°C` when they agree).
- **`log_pool(p_model, p_market, w, modal_debias=True)`:**
  - Caps the modal attenuation on the market's favourite bucket $j^* = \arg\max_j p_{j,\text{mkt}}$ when $j^* \ne \arg\max_j p_{j,\text{mod}}$ to at most $w(1-w)(p_{j^*,\text{mkt}} - p_{j^*,\text{mod}})$, eliminating the second-order Jensen concavity penalty $-w^2\varepsilon^2 / (2\sigma^2)$ that manufactured false `NO` edges across `193` live-rule favourites.

### 3. `CO-003C` — Empirical-Bayes Shrinkage, Glosten-Milgrom Maker EV, Microstructure Veto, and Uncertainty Kelly (`polybot/polybot/strategy.py`)
- **`shrunk_side_prob(p_pool_side, p_mkt_side, kind, lead, cfg)`:**
  - Shrinks the post-pool divergence toward the de-vigged market prior:
    $$p_{\text{eb,side}} = p_{\text{mkt,side}} + \kappa_{\text{eb}}(\text{kind}, \text{lead}) \cdot (p_{\text{pool,side}} - p_{\text{mkt,side}})$$
    with $\kappa_{\text{eb}} = 0.15$ on `highest` at `lead <= 1` and $\kappa_{\text{eb}} = 0.55$ on `highest` at `lead >= 2`.
- **`execution_conditional_prob(p_uncond, adverse_fill_ratio=1.85)`:**
  - Applies Bayes' rule for passive limit orders under adverse selection ([Glosten & Milgrom 1985](https://linkinghub.elsevier.com/retrieve/pii/0304405X85900443)):
    $$p_{\text{cond}} = \mathbb{P}(Y = 1 \mid \text{Filled} = 1) = \frac{p_{\text{eb}}}{p_{\text{eb}} + \lambda (1 - p_{\text{eb}})}$$
    At $\lambda = 1.85$, an unconditional `71.1%` win probability (`f312e9d5ff` pool mean) maps to a fill-conditioned win probability of **`57.06%`**, preventing negative-EV passive bids from being posted unless they carry sufficient spread cushion to survive adverse fill selection.
- **`passive_microstructure_veto(ev, m, side, by, bn, fc_mean_u, con, cfg)`:**
  - Blocks passive `NO` orders when top-3 complement `YES` bid volume exceeds `72%` of top-3 `YES` book volume (`56/108` fills in `f312e9d5ff` were triggered by complement `YES` takers).
  - Blocks same-day (`lead <= 0`, `local_hour >= 10`) passive `NO` orders on `highest` when observed METAR temperatures project a daily high $\ge 0.8^\circ\text{C}$ warmer than `fc_mean` toward the target bucket (the exact Oct 28 Beijing/Busan dry offshore-wind failure mode).
- **`kelly_fraction(p, price, fee_rate=0.0, p_model_side=None, p_mkt_side=None)`:**
  - Uses fee-inclusive effective price $a_{\text{eff}} = a + r_{\text{fee}} a(1-a)$ and multiplies by the [Baker & McHale (2013)](https://pubsonline.informs.org/doi/10.1287/deca.2013.0271) uncertainty shrinkage factor $\frac{\text{edge}^2}{\text{edge}^2 + \sigma_p^2}$ where $\sigma_p^2 = 0.0025 + 0.25(p_{\text{model}} - p_{\text{market}})^2$.

### 4. `CO-003D` — Fix `follow.backtest` and Block Same-Event Correlated Legs (`polybot/polybot/follow.py`, `polybot/polybot/db.py`)
- Moves `obs_only = bool(getattr(cfg, "follow_obs_only", True)) and not day_over` **above** the `if day_over / elif obs_only / else` branch in `follow.backtest` so `obs_only` mode does not call or gate on `future_extreme`, and passes `exact=(ev.get("rule") == "metar")` to `feasible_range`.
- Adds `db.has_any_position(con, market_id)` to prevent `follow` and `forecast` from buying the same or opposite sides of the same `market_id`.
- Adds `db.has_conflicting_event_direction(con, event_id, lo, hi, side, "follow")` to block `follow` from holding both `YES` and `NO` legs in the same event (preventing the Kuala Lumpur `31°C YES + 32°C NO` and Wuhan `26°C YES + 27°C NO` correlated double-losses).

### 5. `CO-003E` — Left-Censored Gaussian in `intraday.py` and Date-Blocked CV in `fit.py`
- Replaces `intraday.bucket_probs`'s left-truncated normal `(cdf(b) - cdf(a)) / (1 - cdf(floor))` with the exact **left-censored normal law** ([Tobin 1958](https://doi.org/10.2307/1907382)): the bucket containing `obs_rounded` receives the point mass $\Phi((\text{floor}-\mu)/\sigma)$ plus continuous mass $\Phi((b-\mu)/\sigma) - \Phi((\text{floor}-\mu)/\sigma) = \Phi((b-\mu)/\sigma)$, while buckets strictly above `obs_rounded` receive uninflated $\Phi((b-\mu)/\sigma) - \Phi((a-\mu)/\sigma)$.
- Replaces `folds = [cases[0::2], cases[1::2]]` in `fit.py` with chronological date-blocked folds (`early_dates` vs `late_dates`) so same-date synoptic weather errors cannot leak across train and test folds.

### 6. `CO-003F` — Stake-Weighted Effective Dates $G_{\text{eff}}$ and CR2/Moulton Cluster Band (`polybot/polybot/governance.py`)
- Adds `effective_dates(dates, stakes)` computing $G_{\text{eff}} = \frac{(\sum_g S_g)^2}{\sum_g S_g^2}$ ([Carter, Schnepel, & Steigerwald 2017](https://ideas.repec.org/a/tpr/restat/v99y2017i4p698-709.html)) and exposes `eff_dates` in `scorecard()`.
- Replaces `_cluster_se` with the Bell-McCaffrey CR2 leverage-corrected ratio-estimator variance ($\tilde{E}_g = (1 - S_g/S_{\text{tot}})^{-1/2} \sum_{i \in g} w_i (r_i - \widehat{\text{ROI}})$), floored by the [Moulton (1986)](https://www.sciencedirect.com/science/article/pii/0304407686900217) intra-date correlation factor $\sqrt{1 + \rho_{\text{date}}(\bar{m}_{\text{eff}} - 1)}$ ($\rho_{\text{date}} = 0.15$) and scaled by the small-sample Student-$t$ multiplier with $\text{df} = \max(1, G_{\text{eff}} - 1)$.

---

## Apply

From `<records>`:

```bash
patch -p1 --dry-run < change_orders/CO-003.diff
patch -p1 < change_orders/CO-003.diff
```

- **During the current freeze (until `2026-11-01T16:23Z`):** Leave `forecast_pause_entries: bool = True` (the default in `CO-003.diff`). This implements Option 2 from `ANALYSIS_forecast_2026-10-30.md`: `forecast` stops opening new paper positions immediately (risk-reducing, permitted under freeze rules) while continuing to log `forecasts` and `snapshots` on every scan.
- **When ready to start the new calibrated `forecast` epoch:** Set `forecast_pause_entries = False` in `config.py` (or `config.json`). Because `forecast_version` is bumped to `"2026-10-30-co003-calibrated"`, `governance.py` will automatically record a clean epoch boundary.

---

## Verify

```bash
cd <records>/polybot
python3 -m py_compile polybot/config.py polybot/probability.py polybot/strategy.py polybot/follow.py polybot/db.py polybot/intraday.py polybot/fit.py polybot/governance.py
python3 -c '
from polybot import probability, strategy, intraday, governance, config
cfg = config.Config()
# 1. Verify de-vigging + modal-debiased log pool
books_y = [{"bids": [], "asks": [[0.02, 100]]}] * 4 + [{"bids": [[0.42, 100]], "asks": [[0.46, 100]]}] + [{"bids": [], "asks": [[0.02, 100]]}] * 4
p_mkt, ok = probability.devig_market_probs(books_y)
assert ok and p_mkt[4] > 0.44, f"devig failed: {p_mkt[4]}"
# 2. Verify Glosten-Milgrom execution-conditional prob (0.711 -> ~0.571 at lambda=1.85)
p_cond = strategy.execution_conditional_prob(0.7113, cfg.forecast_adverse_fill_ratio)
assert 0.56 < p_cond < 0.58, f"p_cond failed: {p_cond}"
# 3. Verify left-censored intraday Gaussian concentrates mass at floor when mu << obs_max
buckets = [(10, 10), (11, 11), (12, 12), (13, 13)]
p_int = intraday.bucket_probs(mu_c=9.0, sigma_c=1.0, obs_max_c=12.0, buckets=buckets, unit="C")
assert p_int[2] > 0.99, f"intraday censoring failed: {p_int}"
# 4. Verify CR2/Moulton cluster SE is strictly larger than unclustered SE on skewed dates (24 trades on D1, 6 on D2)
rets = [0.25]*18 + [-1.0]*6 + [0.25]*3 + [-1.0]*3
dates = ["2026-10-24"]*24 + ["2026-10-25"]*6
stakes = [10.0]*30
assert governance.effective_dates(dates, stakes) < 2.0
assert governance._cluster_se(rets, dates, stakes) > 0.14
print("CO-003 verification passed.")
'
```

---

## Rollback

```bash
cd <records>
patch -p1 -R < change_orders/CO-003.diff
```

---

## CHANGELOG entry

Append to `polybot/notes/CHANGELOG.md`:

```markdown
## 2026-10-30 — CO-003: `forecast` calibration & execution guards, `follow` correlated-leg guard, `intraday` left-censoring, CR2/Moulton cluster governance

- **Change (`config.py`, `probability.py`, `strategy.py`):** Added `forecast_pause_entries = True` (freeze-safe shadow mode), spread-weighted power de-vigging (`devig_market_probs`), EMOS spread-skill kernel dressing (`bucket_probs_emos`), modal-concavity-debiased log pooling (`log_pool(..., modal_debias=True)`), Empirical-Bayes divergence shrinkage (`shrunk_side_prob`), Glosten-Milgrom execution-conditional maker probability (`execution_conditional_prob`, $\lambda=1.85$), passive microstructure/METAR warming vetoes (`passive_microstructure_veto`), and Baker-McHale uncertainty-shrunk Kelly sizing. Bumped `forecast_version` to `"2026-10-30-co003-calibrated"`.
- **Change (`follow.py`, `db.py`):** Fixed `follow.backtest` so `obs_only` mode does not gate on `future_extreme` and passes `exact=(rule=="metar")`. Added `db.has_any_position` (cross-strategy overlap block) and `db.has_conflicting_event_direction` (blocks holding both `YES` and `NO` across buckets of the same event).
- **Change (`intraday.py`, `fit.py`):** Replaced left-truncated Gaussian in `intraday.bucket_probs` with the exact left-censored Gaussian law for $M_{24} = \max(M_t, X_{\text{rem}})$. Replaced alternating-index CV split (`cases[0::2]`) in `fit.py` with chronological date-blocked folds.
- **Change (`governance.py`):** Added stake-weighted effective date clusters `effective_dates` ($G_{\text{eff}} = (\sum S_g)^2 / \sum S_g^2$) and replaced `_cluster_se` with a Bell-McCaffrey CR2 leverage-corrected, Moulton-floored ($\rho_{\text{date}}=0.15$), Student-$t$ scaled cluster SE.
- **Reason:** Across 190 settled `forecast` trades (`d5e06839b5` + `f312e9d5ff`), pool-implied wins (`132.9`, `69.95%`) overstated actual wins (`111`, `58.42%`, $p=0.00084$) while market-implied wins (`109.2`, `57.47%`) matched reality ($p=0.826$). Passive bids in `f312e9d5ff` suffered a $1.853\times$ loser-to-winner adverse fill ratio ($p=0.00347$). `follow` lost `-$19.98` on two correlated adjacent-bucket double-losses on Oct 25, and `CO-002`'s unadjusted `_cluster_se` was smaller than the unclustered SE across all 7 epochs.
```

---


## Full Unified Diff

_Removed from this public copy: the repository holds documents only, not the bot's code._
