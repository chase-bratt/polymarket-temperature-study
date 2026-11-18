# Why forecast is doing worse (2026-10-30)

## The numbers

| forecast settings | fills | settled | won | price implied | our pool implied | P/L |
|---|---|---|---|---|---|---|
| d5e06839b5, Oct 21–25 (paid the ask) | taker | 137 | 83 | 79.4 | 95.2 | **+$16.90 (+4.1% ± 15.5%)** |
| f312e9d5ff, since Oct 25 16:23Z (passive bids, filled only by real prints) | maker | 53 | 28 | 29.8 | 37.7 | **−$13.67 (−10.0% ± 44.9%)** |

New epoch by date: Oct 27 1/1 (+$1.73), **Oct 28 12/30 won vs 16.7 implied (−$19.76)**, Oct 29 15/22 vs 12.5 (+$4.36).
55 positions still open ($153). 2.1 effective dates. Statistically this is not a verdict yet in either direction.

## The hypothesis was "we dropped the ~30% of trades that wouldn't have filled". The data says the opposite.

Passive bids don't remove bad trades. **They keep the bad trades and miss the good ones.**

Every forecast bid since the switch, one row per market and side, on resolved markets:

| | n | won | price implied | P/L at our bid |
|---|---|---|---|---|
| bid got filled | 53 | 28 (53%) | 30.5 | **−14%** |
| bid never filled | 70 | 55 (79%) | 43.7 | **+25%** (counterfactual) |

The pattern holds within each date (Oct 28: filled 12/30 vs unfilled 32/41; Oct 29: 15/22 vs 21/27).

Mechanism, measured on our side's mid 3 h after each bid was posted:

- unfilled bids: price moved **+2.6¢ our way** on average. The market went toward us and left our bid behind.
- filled bids: price moved −0.1¢. Half moved against us.
- 56 of 108 fills were triggered by someone **buying the favourite YES** at the complement price.

That is adverse selection, the winner's curse of resting orders. The people who trade into a resting bid usually know at
least as much as the bid does. When the weather goes our way, nobody sells to us cheaply. When it goes against us, they do.

## The deeper problem: the model overstates our side

In both settings, actual wins track the **market price**, not our pool:

- old: won 83, price said 79.4, pool said 95.2
- new: won 28, price said 29.8, pool said 37.7

The pool overstates the probability of the side we buy by roughly 15–25%. The trade-free test on every stored scan
(Oct 25) said the same thing: favourites are fairly priced. The daily review (Oct 29) found the live rule's favourites won
84/193 vs 84.7 implied by the NO price and 61.6 by our pool. Forecast's "edge" is mostly the pool's overconfidence.

So the old +4% was:
1. within noise (band ±15.5%, 4 effective dates);
2. flattered by paying stale asks. The Oct 25 fill check already said so: paper +$19.76, fillable +$13.76, **confirmed-only
   −$2.61**. The new broker books only the confirmed kind, and −10% is in line with that.

The real-print accounting didn't make forecast worse. It stopped hiding how forecast does.

## What this means

- **Passive execution doesn't rescue a strategy without an edge.** It makes the lack of one show up faster. A passive
  strategy only works when the price you rest at is already favourable before anyone trades into it.
- The "+25% on unfilled bids" can't be harvested. Chasing those bets means paying the ask, which is what the old epoch
  did: +4% on paper, about zero on confirmed fills.
- Wait for 15+ effective dates before a verdict (Oct 28 was one bad weather day: clear, dry offshore-wind highs ran
  1.5–3 °C above our mean in Beijing and Busan while the market priced them right). The evidence so far points one way,
  though: the market is calibrated, and our pool isn't better than it.

Options (Chase's call; all allowed during the freeze):
1. **Keep collecting** to 15 effective dates (about 2–3 weeks), then let the kill rule decide. Cost: small, paper money only.
2. **Pause forecast entries now**: risk-reducing, allowed in the freeze. Keep the snapshots/shadow running for research.
3. **Shrink stakes** (risk-reducing) while it collects.

Not recommended: changing the model or the fill rule mid-freeze to make the number look better.

## Files

- `since_2026-10-25_0920/`: everything since the last export (Oct 25 09:20Z): fills (121), settled trades (176),
  open positions (55), every resting bid with its outcome (725), every decision incl. rejections (5,220), trade reviews.
- `performance_all_time/`: the full performance workbook (12 sheets) and CSVs, regenerated today.
- `strategy/`: decision code (strategy, execution, follow, probability, forecast, fit, governance, scoring…), config,
  STRATEGY.md, PREREGISTRATION.md, CHANGELOG.md, scoring manifest, all daily/weekly reviews, the latest official scorecard.
