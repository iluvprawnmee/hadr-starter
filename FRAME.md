---
shaping: true
---

# HADR Monitor — Frame

The "why" behind the HADR Monitor, distilled before solution work. Ground truth
for the problem; `SHAPING.md` explores the how.

## Source

Verbatim from `README.md` ("The end state"):

> By Wednesday afternoon this repository contains an agent that:
> - watches live disaster feeds — GDACS, USGS and ReliefWeb (see `feeds/`)
> - filters out the noise and assesses what remains: what happened, where, how bad, who is affected
> - publishes a morning situation report to `dashboard.html` at 08:30 Singapore time
> - runs on a schedule, unattended, and stays quiet when nothing has changed

Verbatim from `REQS.md` (the idea in one line):

> An unattended agent that watches three live disaster feeds, decides what is
> genuinely *new* since it last reported, and publishes a ranked morning situation
> report to `dashboard.html` at 08:30 Singapore time — staying quiet (but not
> silent) when nothing has changed.

Verbatim from `docs/blindspots.md` (the framing that matters):

> The product is a **diff engine, not a fetcher**. Fetching is trivial; the system
> is the editorial policy that decides what counts as news, backed by a persistent
> event store.

## Problem

A humanitarian duty analyst anchored to Singapore time has no trustworthy way to
know, each morning, which natural-hazard events in Asia-Pacific are genuinely new
or changed since the last briefing. Doing it by hand is impossible: GDACS Green
noise is ~95% of volume, USGS is dominated by tiny instrument-dense quakes, and
ReliefWeb posts hundreds of items a day. The feeds actively work against a naive
solution — they are rolling windows with no history and no deletion tombstones,
carry three incompatible time formats, and expose impact signals (PAGER, GDACS
colour) that arrive late and get revised. The result: an analyst cannot tell
"quiet" from "crashed," cannot trust a report built on mis-parsed timestamps, and
never learns when yesterday's headline event was quietly downgraded or deleted.

## Outcome

Every morning at 08:30 SGT, the analyst opens one ranked situation report that
they can trust:

- It shows only what is genuinely new or changed since the last report — no
  re-run of yesterday's news, no Green-level spam.
- It reads as one story per real-world crisis, even when several feeds and
  multiple events describe it.
- It is honest about its own state: visibly quiet (not silent) on a calm morning,
  banner-flagged when a feed is down, and it issues corrections when a prior
  headline is downgraded or deleted.
- It never silently drops a genuinely dangerous event, and never lies about
  timing.
- It runs unattended, survives a missed run, and can be replayed deterministically
  for review and demo.
