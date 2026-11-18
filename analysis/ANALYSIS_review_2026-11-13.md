# polybot review of the 2026-11-13 export, and guidance on the ML restructure

Source: `data/raw/polybot_2026-11-13/` (full raw tables, 2026-10-18 to 2026-11-13 10:40Z) and
`snapshots/polybot_export_2026-11-13/` (notes, code, CHANGE_REGISTER). Every number below was recomputed from the raw
parquet with duckdb. Nothing in `data/` or `snapshots/` was modified.

## Who acts on what

Two systems. The **Hermes cloud agent** runs the live bot and owns all changes to the bot and its strategy. The **local
machine** holds the datasets and builds every ML model (temperature model, log-pool blend, execution model). Tags below:

- **[HERMES]** — the cloud agent acts on this.
- **[LOCAL]** — done on the local machine. Hermes ignores it.
- **[LOCAL → HERMES]** — built and validated locally; Hermes acts only when a validated model or rule is handed over.
- **[CONTEXT]** — findings only, no action for anyone.

**Hermes, act on these now:**

1. Keep capturing; change nothing about collection (5.1).
2. Resolve the disk question PC-15 this week; the volume fills around 11-28 (5.2).
3. Drop the queued tuning items PC-02, PC-06, PC-14 (5.3).
4. Keep the confirmed-print fill rule (5.4).
5. Do not drop YES bids; do not change follow (end of 5).

Everything else is either context or local work. No strategy change from sections 4 or 6 goes live until a model is
validated locally and handed over.

## 1. Where the bot stands [CONTEXT]

| forecast era | settled | won | price implied | pool implied | P/L | ROI | dates |
|---|---|---|---|---|---|---|---|
| pre-pool (to 10-21) | 249 | 78 | 81.7 | 118.3 | −$182.88 | −29.9% | 5 |
| pool, taker fills (10-21 to 10-25) | 137 | 83 | 79.4 | 95.2 | +$16.90 | +4.1% | 5 |
| passive bids (10-25 to 10-30) | 113 | 57 | 63.8 | 80.6 | −$34.08 | −11.2% | 6 |
| **current: fresh runs + passive (since 10-30)** | **295** | **168** | **162.7** | **205.2** | **−$2.13** | **−0.3%** | 13 (11.5 eff.) |

- The current epoch is break-even: 168 wins against 162.7 implied by the prices paid. The pool expected 205. Same
  pattern as every earlier epoch: wins track the price, not the pool. The register's numbers reproduce exactly.
- By side, current epoch: NO 277 trades, 167 won, +$19.46. YES 18 trades, 1 won (2.7 implied), −$21.60. Not significant
  on its own (p ≈ 0.2), but it matches the history (YES under 5¢ won 1 of 133; early longshots 0 of 17).
- Adverse selection persists. Bids that filled: 168 of 295 won, +1.4¢ per share over the bid. Bids that never filled:
  246 of 362 would have won, +7.3¢ per share. Outage window excluded.
- follow: 0 settled since 10-30. Still dormant.
- Data health: order-book files decode (1,043 full-book scans, 4,027 five-minute deltas). One 11.5 h gap on 10-27 (the
  known polyv2 crash) and the trade-print outage 11-02 22:10Z to 11-04 03:31Z. Labels: 2,234 resolved events, 25 dates.

## 2. The number that matters for the restructure: model vs market [CONTEXT]

Log-loss on the bucket that actually won (lower is better), liquid books, events since 10-23, one value per event:

| kind, lead | events | dates | our model | market (no-bid buckets at ask/2) | pool | model better than market on | pool better than market on |
|---|---|---|---|---|---|---|---|
| highs, lead 1 | 989 | 21 | 1.569 | **1.279** | 1.317 | 0 of 21 dates | 6 of 21 |
| highs, lead 2 | 909 | 20 | 1.624 | 1.512 | **1.479** | 3 of 20 | **15 of 20** |
| lows, lead 1 | 302 | 21 | 1.784 | **1.576** | 1.659 | 2 of 21 | 3 of 21 |
| lows, lead 2 | 133 | 20 | 1.911 | **1.785** | 1.813 | 7 of 20 | 8 of 20 |

- One day ahead the market beats the weather model on every single date. The gap is 0.28 ± 0.06 nats.
- The only place the bot adds information is highs two days ahead: pool minus market −0.034 ± 0.020 nats, better on 15
  of 20 dates. That is where the market is least informed, and where a better temperature model can pay.
- This table is the scoreboard for the new temperature network. "Lower RMSE than before" is not the goal. "Lower
  log-loss than the market's price on dates the model never saw" is.

## 3. How good is the temperature forecast already [CONTEXT]

RMSE of the daily maximum against the station, °C:

| predictor | RMSE |
|---|---|
| station-month climatology (2-year history) | 3.76 |
| persistence: yesterday's maximum | 3.22 |
| one global model, lead 1 (GFS 2.00, ECMWF 1.91) | ~1.95 |
| mean of ECMWF + GFS, lead 1 (history product) | 1.69 |
| **live 140-member ensemble + deterministic blend, lead 1** | **~1.15** (1.18 measured against 1° buckets) |
| live blend, lead 2 | ~1.25 |

Physics models already remove 70% of the error that climatology and persistence leave. A network trained on past
weather alone has to rediscover atmospheric physics from 50 stations; it will land near persistence (about 3 °C), not
near 1.15 °C. The network's job is to correct the forecast, not replace it.

## 4. Guidance on the plan [LOCAL]

All of section 4 is local ML work. Hermes ignores it. The only handover points are the validated outputs at the end of
the 4.5 workflow (blend weights, execution model, paper-trade rules), which come to Hermes as [LOCAL → HERMES].

### 4.1 What the 1960-to-now download is, and is not [LOCAL]

- Open-Meteo's historical weather back to 1960 is **reanalysis** (ERA5 / ERA5-Land): a model's reconstruction on a
  9 to 25 km grid. It is not the airport thermometer the markets resolve on, and it is not a forecast.
- Archived **forecasts** do not go back that far. Open-Meteo's historical forecast archive starts around 2021-2022, and
  lead-specific "previous runs" start in 2024 (this is what `history/fc_daily` already holds). Verify the exact start
  dates in their docs before planning around them.
- So the long history cannot teach "given this forecast, what happens": there is no forecast in it. It is still
  useful for three things: station climatology and anomalies as input features, the grid-versus-station offset by
  season and regime, and pre-training. For labels you need the **station** record (IEM / NOAA ISD hourly METARs go back
  decades), resolved with the same rule as the market.
- The 1960s climate is about 1 °C cooler and stations moved or changed instruments. Use anomalies against a rolling
  normal, or weight recent years.

### 4.2 Frame the temperature model as forecast post-processing [LOCAL]

- **Target:** the error of the forecast (station observation minus forecast), or directly the probability of each
  integer reading. Not the raw temperature.
- **Inputs, all known at decision time:** ensemble mean and spread, disagreement between models, forecast cloud,
  dewpoint, wind direction and speed, lead, station, day of year, the station's recent forecast errors, climatological
  anomaly. The bot's own loss reviews point at cloud and dewpoint regimes (dry clear days run warm, stratus days run
  cold): those are the first features to add.
- **Output:** a distribution, not a point. The market pays on buckets. Train and score with a probabilistic loss
  (log-loss on the bucket, or CRPS), and check calibration.
- **Usable training size:** forecast-plus-outcome pairs. About 37,000 station-days at 2 years (existing backfill),
  perhaps 90,000 if the 2021+ archive is added. That is the real dataset, not 66 years.

### 4.3 The market network is not trainable yet [LOCAL; the fitted blend and execution model → HERMES once validated]

- There are 25 resolved dates. All cities on one date share one weather regime and one market mood, so the effective
  sample is about 25, not 2,234 events or 2.5 million snapshots. A neural network on that will memorise it.
- The market is already calibrated (favourites won 1,063 of 2,172 against 1,022 implied). There is no general
  mispricing to learn. The only question is whether your temperature distribution adds information to the price, and
  that is a two- or three-parameter blend (the log pool), fitted by date-blocked cross-validation.
- What the market data can support now is an **execution model**: 5,420 resting bids with fill or no fill and the
  later outcome. Predicting fill probability and the outcome conditional on being filled (the adverse-selection cost)
  is a well-posed supervised problem with usable volume. Start with logistic regression or gradient-boosted trees.
- Revisit a learned market model at 100+ resolved dates. Until then every extra day of capture is worth more than any
  architecture choice.
- *(Added after review.)* Possible shortcut to more dates: Polymarket trades settle on-chain, so trade and price history
  for past resolved weather markets should be public (verify). Each backfilled date with prices and a known outcome adds
  one resolved date for the log-pool blend. It does **not** give the full order book (resting bids that never filled,
  queue position), so it does not help the execution model. Masked replay of the temperature model on past dates is
  just ordinary train/test on history; it cannot manufacture market dates.

### 4.4 Splitting and evaluation [LOCAL]

- Three sets, not two: **train**, **validation** (for every tuning decision), **test** (opened once, at the end).
  Tuning on the test set turns it into a second training set.
- Split by **time**, in whole dates, never by random row. Example for the history: train to 2024, validate 2025,
  test 2026. For the market data: walk forward, always predicting dates later than anything trained on.
- Compute scaling statistics, climatologies and station biases on the training period only.
- Build a per-row "known at" timestamp and enforce it: a forecast run counts only after its availability time
  (`model_runs`), an observation only after `metar_seen.first_seen`.
- Keep the **baseline ladder** and only climb when the next rung wins out of sample:
  climatology → persistence → raw ensemble → linear correction (EMOS) → gradient-boosted trees → neural network.
  On tabular data of this size, boosted trees usually match or beat a network and take a tenth of the effort. A
  network earns its place if it beats them on the validation years.

### 4.5 Recommended workflow [LOCAL until "paper trade", which is → HERMES]

```
 station METARs (IEM/ISD)     archived forecasts 2021+      reanalysis 1960+        bot ledger (since 10-18)
 = LABELS, market rule        + live ensemble = INPUTS      = climatology features  prices, books, bids, fills
          │                            │                            │                         │
          └──────────────┬─────────────┴──────────────┬─────────────┘                         │
                         ▼                            ▼                                       │
              one row per station × date × lead, every column stamped "known at"              │
                         │                                                                    │
                         ▼                                                                    │
        split by DATE:  train ──► validation ──► test (opened once)                           │
                         │                                                                    │
                         ▼                                                                    │
        baseline ladder: climatology → persistence → raw ensemble → EMOS → trees → network    │
                         │                                                                    │
                         ▼                                                                    ▼
        bucket probabilities p_model ──────────────► blend with market price (2-3 parameters, date-blocked CV)
                                                                    │
                                                                    ▼
                                         score: log-loss vs market on unseen dates  (today: 1.57 vs 1.28)
                                                                    │
                                              beats market? ── no ──► do not trade; keep collecting
                                                                    │ yes
                                                                    ▼
                                         execution model (fill probability, adverse selection) → paper trade
```

## 5. Changes worth making now, given the restructure

1. **[HERMES] Keep the bot capturing, untouched.** Market dates are the scarce resource and only the calendar produces them.
   Do not stop collection while rebuilding.
2. **[HERMES] Decide the disk question (PC-15) this week.** The ledger grows about 90 MB a day and the volume fills around
   11-28. A full disk stops the one thing that cannot be re-downloaded.
3. **[HERMES] Do not spend effort on the queued tuning items (PC-02, PC-06, PC-14).** They tune a model you are
   replacing. **[LOCAL]** Their evidence carries over as design input for the new model: dispersion too narrow, pool
   weight too high at lead 1.
4. **Carry these into the new design:** **[LOCAL]** score against the market with no-bid buckets at ask/2 (the stored
   `p_market` understates favourites), keep date-blocked evaluation and effective dates. **[HERMES]** keep the
   confirmed-print fill rule.
5. **[LOCAL] Aim the first model at highs two days ahead.** It is the only cell where the current system already beats the
   market, so it is where an improvement is most likely to be worth money.

**[HERMES]** Not recommended on this evidence: dropping YES bids (suggestive, not significant), any change to follow.

## 6. Edges to test (added after review, not computed from the data) [LOCAL first; Hermes only on handover]

Everything in sections 1 to 5 was measured. This section was not: these are hypotheses from discussing the review, to be
tested against the ledger before any build. The market is calibrated on average, so profit has to come from a specific,
measurable gap. Three candidates:

1. **[LOCAL → HERMES] Forecast edge, highs at lead 2 only.** The one measured edge (section 2): pool beats market on 15
   of 20 dates, by −0.034 ± 0.020 nats. Thin. Restrict to lead 2. Lead 3+ was never measured; do not extend to it
   without running the same log-loss-vs-market table first. Validated locally; Hermes changes nothing until handed a
   model that beats the market on unseen dates.
2. **Speed edge.** **[LOCAL]** the lag test below. **[LOCAL → HERMES]** the source feeds and event-driven loop, only if
   the test shows a gap of minutes. The market may be slow for minutes after new information lands. Open-Meteo is downstream of the
   model centres, so polling it every 5 to 10 minutes cannot win a speed race; a speed edge needs source data (raw model
   runs, e.g. GFS from NOAA as each run publishes; live METARs) and an event-driven loop that reacts on arrival, not on a
   timer. Late on settlement day the high is often effectively locked in by the METARs.
   **Test first, build second:** join `metar_seen.first_seen` and `model_runs` availability times to the price history
   and measure how long prices take to move after each. Seconds: someone faster owns it, drop it. Minutes: worth building.
3. **Spread / market making.** **[LOCAL]** the execution model. **[LOCAL → HERMES]** any two-sided quoting, after the
   execution model is validated. The passive bids are already one side of this. The cost is adverse selection: filled bids
   earned +1.4¢ per share, unfilled ones would have earned +7.3¢ (section 1), i.e. fills arrive when the counterparty
   knows more. Two-sided quoting is untested. It depends on the execution model (4.3) predicting which bids get picked
   off, so they can be pulled or repriced first.

**Order:** forecast edge at lead 2 first (it gates trading at all, per the 4.5 workflow), then the speed-lag test (cheap,
uses data already captured), then the execution model, then any two-sided quoting on top. Paper trade each before money.
