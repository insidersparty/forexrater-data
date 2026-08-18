# forexrater-data

Broker-published spread samples collected by the ForexRater Spread Observatory.

## What this is, and what it is not

Every number here is a quote **published by the broker on its own public endpoint**
and recorded as-is. Nothing is measured by us in the sense of trading it: we do not
hold accounts, place orders, or observe fills. `source` is always
`broker-published`, never `measured`.

`feedType` records whether the broker states these are tradeable or indicative
prices. Where a broker states nothing it is `unspecified`, and that is what the
site displays — not a guess.

**A gap is data.** If a sample is missing, the request failed or the payload did not
contain that symbol. Nothing is interpolated, carried forward, or synthesised. A
window that fails a quality gate is written to `quarantine/` and never served.

**A held window keeps its own date.** `latest-replay/` is republished per window, and
only when the QUOTE CONTENT changed. A window whose bid/ask/spread sequence is identical
to the one already published is left untouched, so it continues to show the date it was
actually recorded rather than being relabelled with today's. This means different symbols
can legitimately carry different dates — that is the honest state, not a synchronisation
bug. Rewriting an unchanged window under a fresh timestamp would present a frozen feed as
new evidence, which is the one thing this repo exists to avoid.

## Repository notes

Authorship metadata corrected 2026-08-18, prior to public launch; commit contents
unchanged. The rewrite touched author and committer fields only — every tree, blob,
message and timestamp in this repository is byte-identical to what it was before.

## Layout

    spreads/{date}/{broker}-{symbol}-{session}.csv   one row per sample
    latest-replay/{broker}-{symbol}.json             most recent passing window
    quarantine/{date}-{broker}-{symbol}-{session}.json   rejected, kept for diagnosis

## Quality gates

Both must pass or the window is quarantined and the run goes red.

**1. Spread plausibility.** The window MEDIAN must be within 0.3x-4x the broker's
advertised figure when that figure is >= 0.3 pips; below that the ratio degenerates
(an advertised "0.0 pips" makes every ratio undefined or zero), so a per-symbol
absolute band applies instead:

| Symbol | Band | Unit |
|---|---|---|
| EUR/USD | 0.0 – 3.0 | pips |
| GBP/USD | 0.0 – 4.0 | pips |
| USD/JPY | 0.0 – 3.5 | pips |
| XAU/USD | 0.02 – 6.00 | USD |

A window where every sample is identical AND zero is quarantined: that is
indistinguishable from a parse silently producing 0.

**2. Mid-price sanity.** The median mid ((bid+ask)/2) must fall inside a decade-scale
range per symbol. The spread bands cannot see a wrong-magnitude *price* — a
divide-by-ten error or a JPY pair parsed with 4 decimals yields a plausible-looking
spread over a nonsense price.

| Symbol | Mid range |
|---|---|
| EUR/USD | 0.8 – 1.6 |
| GBP/USD | 1.0 – 1.8 |
| USD/JPY | 80 – 250 |
| XAU/USD | 1000 – 10000 |

## Display demotion (collection is unaffected)

A series whose window median is **0.0** on an instrument where zero is not a plausible
tradeable cost is **display-demoted**: it stops ticking on the site and renders as the
broker's advertised row, flagged *"sampled feed under review"*.

Collection does not change. It is still sampled, still committed here, still counted in
the health table below — the record is honest and hiding it would be worse. The demotion
is about presentation: animating a row that reads 0.00 throughout would present a number
nobody could trade as though it were a live cost.

The quality gates cannot catch this case. Such a window is internally consistent and
passes both the band and the mid-price check; the problem is what the number *means*,
not whether it parsed. The rule is general, not a per-broker exception — a series
reinstates itself automatically once its medians become plausible, or when the broker
documents the feed.

Currently demoted: **pepperstone / EURUSD** (publishes bid == ask on most samples).

## Feed health

Derived from the files in this repo, not hand-maintained. A feed below 70% window
success is flagged for review and demotion to advertised-only display.

| Broker | Symbol | Committed | Quarantined | Success | Last good | Status |
|---|---|---|---|---|---|---|
| dukascopy | ETHUSD | 7 | 0 | 100% | 2026-08-18 | ok |
| dukascopy | EURUSD | 11 | 0 | 100% | 2026-08-18 | ok |
| dukascopy | GBPUSD | 11 | 0 | 100% | 2026-08-18 | ok |
| dukascopy | USDJPY | 11 | 0 | 100% | 2026-08-18 | ok |
| dukascopy | XAGUSD | 7 | 0 | 100% | 2026-08-18 | ok |
| dukascopy | XAUUSD | 11 | 0 | 100% | 2026-08-18 | ok |
| pepperstone | EURUSD | 5 | 6 | 45% | 2026-08-18 | ⚠ REVIEW |
| pepperstone | GBPUSD | 11 | 0 | 100% | 2026-08-18 | ok |
| pepperstone | USDJPY | 11 | 0 | 100% | 2026-08-18 | ok |
| pepperstone | XAUUSD | 11 | 0 | 100% | 2026-08-18 | ok |

- **pepperstone/EURUSD** — pepperstone/EURUSD (promote-check): every sample identical and zero — indistinguishable from a parse that produced 0. QUARANTINED.

## Collection rules

Public endpoints only; plain unauthenticated fetches; sequential requests 5–10s apart
in bounded session windows, 3x/day; robots.txt fetched first and obeyed; a descriptive
User-Agent with a contact address on every request. A broker that blocks us is
excluded from collection, not worked around.

Generated by `scripts/observatory-readme.mjs`.
