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
    shadow/{date}/{broker}-{symbol}-{session}.csv    held or parked: collected, never served

## Quality gates

Both must pass or the window is quarantined and the run goes red. The single exception
is a **parked** series (below): it is not gated at all, because it is no longer being
published and the gate has nothing left to protect.

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

## Parked series (ruling 2026-08-26)

A **parked** series is one that has stopped being worth gating. It leaves the active
sample set: it is not gated, not quarantined, it cannot turn a sampler run red, and it
accumulates no further published history — which also means it can never reach the
21-day history gate that a broker/symbol must pass before any advertised-vs-sampled
comparison is built for it. It is still fetched, and once a week it is recorded to
`shadow/`, so that a feed which recovers is noticed rather than assumed dead.

Parking is narrower than it sounds, and deliberately so. It is not a softer gate and not
a suppressed failure: the series is no longer published at all, so there is nothing left
for a gate to protect. Everything already collected stays exactly where it is — the
committed windows, the quarantine entries, the counts in the health table below. The
record is not rewritten; only the forward collection stops.

**Currently parked: pepperstone / EURUSD.**

The reason is that the failure is chronic rather than intermittent. Every quarantined
window this series has produced is the same one: an all-identical-and-zero window, bid
== ask on every sample. Across the entire run there are **zero** request failures, zero
parse failures and zero auth failures — the endpoint answers, correctly, with a number
that is not a tradeable cost. There is no fault on our side to fix and no reason to
expect the next window to differ, so a gate that fires on every single run has stopped
carrying information; it only trains the reader to ignore a red run.

Unparking is a human ruling, never automatic. The weekly heartbeat reports the series'
median and its distinct-value count; a person decides whether a change is real.

**Nothing a visitor sees changes.** Pepperstone keeps its sampled rows on the GBP/USD,
USD/JPY and XAU/USD tabs, its advertised row on BTC/USD, and its place in reviews,
comparisons and swap tables. Nothing on the site reads this series' numbers.

The demoted window `latest-replay/pepperstone-EURUSD.json` is nevertheless RETAINED, and
that is deliberate. It is the only thing that puts Pepperstone into the widget's EUR/USD
row list at all: the broker publishes no EUR/USD row of its own for that widget, and its
headline figure of "0.0 pips (Razor)" parses to zero, which the fallback branch skips.
Deleting the file would drop the row from the list permanently rather than demote it.

For accuracy about what is on screen today: that row currently sorts below the tab's
12-row display cap, so Pepperstone is not visible on the EUR/USD tab on either surface —
checked in the rendered DOM on production, and equally true before this ruling. Parking
did not change it and retaining the window does not hide it; the file is kept so the row
returns if the cap or the brokers ahead of it change, instead of being deleted out of the
list for good.

### Display demotion is a separate thing

Demotion is about presentation and applies to a series that is still fully collected and
gated. Parking is about collection. A series can be demoted without being parked; the
Pepperstone EUR/USD feed is now both, having been demoted first and parked later once
the pattern proved permanent.

## Feed health

Derived from the files in this repo, not hand-maintained. A feed below 70% window
success is flagged for review and demotion to advertised-only display. A row marked
`⏸ parked` is no longer collected into `spreads/`: its figures are historical and
frozen, and it is not flagged for a review that has already been held.

| Broker | Symbol | Committed | Quarantined | Success | Last good | Status |
|---|---|---|---|---|---|---|
| dukascopy | ETHUSD | 38 | 0 | 100% | 2026-09-04 | ok |
| dukascopy | EURUSD | 41 | 0 | 100% | 2026-09-04 | ok |
| dukascopy | GBPUSD | 41 | 0 | 100% | 2026-09-04 | ok |
| dukascopy | NAS100 | 38 | 1 | 97% | 2026-09-04 | ok |
| dukascopy | SPX500 | 38 | 1 | 97% | 2026-09-04 | ok |
| dukascopy | US30 | 38 | 1 | 97% | 2026-09-04 | ok |
| dukascopy | USDJPY | 41 | 0 | 100% | 2026-09-04 | ok |
| dukascopy | XAGUSD | 38 | 0 | 100% | 2026-09-04 | ok |
| dukascopy | XAUUSD | 41 | 0 | 100% | 2026-09-04 | ok |
| pepperstone | EURUSD | 14 | 11 | 56% | 2026-08-26 | ⏸ parked |
| pepperstone | GBPUSD | 41 | 0 | 100% | 2026-09-04 | ok |
| pepperstone | USDJPY | 41 | 0 | 100% | 2026-09-04 | ok |
| pepperstone | XAUUSD | 41 | 0 | 100% | 2026-09-04 | ok |

- **dukascopy/NAS100** — NAS: no mid-price range defined — cannot sanity-check magnitude, so not committed
- **dukascopy/SPX500** — SPX: no mid-price range defined — cannot sanity-check magnitude, so not committed
- **dukascopy/US30** — US: no mid-price range defined — cannot sanity-check magnitude, so not committed
- **pepperstone/EURUSD** — pepperstone/EURUSD (promote-check): every sample identical and zero — indistinguishable from a parse that produced 0. QUARANTINED.

## Collection rules

Public endpoints only; plain unauthenticated fetches; sequential requests 5–10s apart
in bounded session windows, 3x/day; robots.txt fetched first and obeyed; a descriptive
User-Agent with a contact address on every request. A broker that blocks us is
excluded from collection, not worked around.

Generated by `scripts/observatory-readme.mjs`.
