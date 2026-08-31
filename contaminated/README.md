# Contaminated windows — withdrawn from the evidence base

These files are NOT deleted and NOT served. They are real recordings, but each one holds
TWO separate sampling windows in a single file, so none of them is the thing its name
claims to be: one window, recorded in one session.

## How it happened

The session label used to be derived from the wall-clock hour a run executed, rather than
from the cron slot that scheduled it. That is correct only while the scheduler is punctual.
From 2026-08-27 GitHub began delivering these runs 350 to 700 minutes late, so two runs
could land in the same derived bucket — and when they did, the second APPENDED to the
first's file.

The 2026-08-14 files are the same failure from the opposite cause: two setup runs that day
both resolved to `newyork`.

The fix, applied 2026-08-31, derives the label from `github.event.schedule` — the cron
expression that actually triggered the run — so a slot six hours late still knows which slot
it is. The sampler additionally refuses to append to an existing window file: a second
window for the same slot and day is written to `collisions/` instead, intact and separate.

## Why they are not split back apart

The rows separate cleanly. Each file holds two clusters of samples about seven seconds
apart, with a gap of hours between them, so cutting them in two is mechanical.

What cannot be recovered is **which scheduled slot each half belonged to**. GitHub exposes
the triggering cron to a running workflow but not through the API afterwards, so for these
historical runs the slot is unsourceable. Splitting them would mean writing a session name
onto real market data that we cannot support — the one thing this archive exists not to do.

So they are withdrawn whole. The 21-day display gate counts windows in `spreads/`, and it
must not open on evidence that merges two recordings into one.

## What is here

- **2026-08-14** — 8 files, each 24 rows spanning 131 minutes: one window at 12:52 UTC,
  another at 15:01 UTC.
- **2026-08-27** — 12 files, each 24 rows spanning 238 minutes: one window at 19:48 UTC,
  another at 23:45 UTC.
- **2026-08-28** — 12 files, each 24 rows spanning 491 minutes: one window at 12:40 UTC,
  another at 20:49 UTC.

Total: 32 files.

## One thing this does not fix

2026-08-27 keeps twelve single-window files labelled `london`. Those windows are clean — one
contiguous recording each — but their label came from the same broken derivation, so the
session name may be wrong even though the data is not. They are left in place and flagged
here rather than relabelled, for the same reason as above: the correct slot is not
sourceable, and a guessed label is worse than a disclosed doubt.
