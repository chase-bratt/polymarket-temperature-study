# Speed-lag test, 2026-11-13: reading of the results

Tables: `data/reports/speed_lag.md` (regenerate with `python pipeline/research_speed_lag.py`). Daily highs, scans and
trades since 2026-10-30 08:00Z, 12 to 15 target dates. All intervals are 95% bootstraps over whole dates.

## Findings

1. **The lag is real two days ahead.** After the bot's forecast jumps by ≥ 0.3 °C, the market's implied mean moves a
   further 54% of the jump over the next 12 h (13% to 97%), 75 jumps on 14 dates. One day ahead the follow-through is
   8% to 14% and mostly done within 6 h.
2. **The market also moves first.** In the 6 h before the bot sees a jump, the market has already moved 59% of it at
   two days (25% to 94%) and 68% at one day (50% to 90%). Other traders have the information earlier.
3. **Takers are paid two days ahead: +3.9% (+1.9% to +6.4%) on $350k.** One day ahead +2.0% (−0.4% to +4.3%). There
   is no sign that takers right after a new ensemble run do better than later ones.
4. **At the bot's current speed the lag is not tradable by crossing the spread.** Buying the bucket the jump favours
   at the ask and selling at the bid 6 or 12 h later loses 3 to 6 cents a share in all four cells, every interval
   below zero. Held to resolution the result is about zero with a ±20% band (28 to 126 trades).
5. **Why 1 and 4 are both true:** a 0.4 °C jump followed 54% by the market shifts the implied mean about 0.2 °C. That
   moves a bucket price by a couple of cents. The spread paid is 5 to 8 cents two days ahead, 3 cents one day ahead.
6. **Open-Meteo publishes GFS 5.9 h and ECMWF 7.7 h after the run starts.** The centres release earlier than that (GFS
   roughly 3.5 to 4.5 h on NOAA's servers; check current schedules). The gap is where finding 2 comes from.

## What this means for the plan

- Speed alone, at today's latency and with today's forecast, does not pay. The follow-through is smaller than the
  spread.
- The edge the takers are collecting is information, not reaction time to Open-Meteo. To be in that group the bot needs
  the half of the move that happens before it currently sees anything: source data from the centres directly, and a
  forecast that is right about what the new run means.
- The lag is still useful on the passive side. A quote that moves on new information, bids in the direction of the
  jump and nothing resting against it, earns the spread instead of paying it. That is the only form of "both sides"
  the data supports, and it needs the execution model.
- Not tested here: the same-day METAR speed edge (section 6.2 of the review), and any strategy using raw model data,
  because none has been captured yet.

## Limits

Two weeks of data. Scans are 30 minutes apart, so reaction faster than that is invisible. The replay sells at the
displayed bid with no size limit. The jump signal is the bot's current forecast, which loses to the market one day
ahead; a better forecast could change finding 4.
