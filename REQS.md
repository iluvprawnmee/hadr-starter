# REQS — HADR Monitor (initial idea capture)

Raw requirements dump for the HADR monitoring agent. Captured 2026-07-08 from an
interview grounded in `README.md`, `feeds/*.md`, and `docs/blindspots.md`. This is
the seed for the planning process (grill-with-docs → PRD → shaping → breadboard);
it is deliberately opinionated but not yet a spec.

## The idea in one line

An unattended agent that watches three live disaster feeds, decides what is
genuinely *new* since it last reported, and publishes a ranked morning situation
report to `dashboard.html` at 08:30 Singapore time — staying quiet (but not
silent) when nothing has changed.

## Why this is hard (the framing that matters)

The product is a **diff engine, not a fetcher**. Fetching is trivial; the system
is the editorial policy that decides what counts as news, backed by a persistent
event store (the feeds are rolling windows with no history and no deletion
tombstones — if we poll and discard, we can never detect change). See
`docs/blindspots.md` for the full blindspot pass; the decisions below answer the
seven policy questions that the feeds themselves cannot.

## Feeds (given)

- **GDACS** — multi-hazard detection (natural hazards only; no landslides/
  epidemics/heatwaves/conflict). Alert levels = modeled humanitarian impact,
  revised as the model reruns. Naive UTC timestamps. Point geometry only.
- **USGS** — earthquakes only. Humanitarian signal is the PAGER `alert` field,
  which lags 20–30+ min and gets revised. `all_day` is instrument-density noise.
- **ReliefWeb** — the `reports` content type (not `disasters`) carries the
  substance: sitreps, casualty figures, appeals. Hard cap 1,000 calls/day.

## Decisions (this interview)

### 1. Scope — Asia-Pacific, natural hazards
Geography anchored to the 08:30 SGT report window: Asia-Pacific region. Natural
hazards only — matching what the three feeds can actually deliver. All-hazard
(conflict/epidemics/industrial) is explicitly out of scope: the feeds can't
deliver it.

### 2. Severity floor — Orange+ / PAGER yellow+ / any ReliefWeb disaster
Report-worthy = GDACS **Orange or Red**, OR USGS PAGER **yellow or above**, OR any
ReliefWeb-declared disaster in region. This is a precision-leaning floor that
keeps the ~95% Green volume and the small-quake instrument noise out.

### 3. Change semantics — what counts as "news"
For an event already known to the store, re-surface it in the morning report on:
- **Escalation** — alert level up (e.g. Orange→Red).
- **Material revision** — magnitude / impact change past a threshold.
- **Corrections** — downgrades AND deletions of an event we previously
  headlined (a credible sitrep issues corrections). Applies even when the change
  is "downward."
- **First ReliefWeb sitrep** on a known disaster.
- **New episode for fast-moving hazards** — a cyclone's updated forecast track is
  news; a static flood re-issue is not. Change logic is hazard-aware.
Explicitly NOT news: Nth sitrep with no new substance, Green-level episode churn,
static re-issues.

### 4. Report window — since last successful report, UTC-normalized
The window covers everything since the last *good* report (not a fixed 24h), so a
missed run never creates a gap. All feed timestamps normalized to UTC at ingest
(GDACS naive strings are UTC; USGS epoch-ms; ReliefWeb RFC-822 — three parsers).
On a feed-down morning the report **still publishes** with a per-feed stale-data
banner ("USGS last fetched HH:MM"), rather than holding.

### 5. Cadence & budget — poll into a persistent store
Poll each feed periodically (interval a PRD tuning detail), archive every raw
payload, and diff the 08:30 report against the persistent store. ReliefWeb call
budget allocated per scheduled run to stay under 1,000/day; no naive
one-call-per-known-disaster loop.

### 6. Testing & demo — fixtures, not live feeds
A deterministic **fixture-replay harness in `scripts/`** records real feed
payloads and replays a "real day" (including a Red event) through the pipeline —
this both de-risks the Wednesday demo (disasters are rare; live feeds may be
boring) and makes the diff engine testable. Plus a hand-crafted **synthetic
"worst-day" fixture**: magnitude revision, event deletion, cyclone episode churn,
and a feed-down — to prove the diff and corrections logic on the nasty cases.

### 7. The report (`dashboard.html`)
- **Quiet day:** still publishes — "No significant changes since [last report]" +
  a heartbeat (feeds healthy, last-fetch times). Quiet ≠ silent, so we can always
  tell "quiet" from "crashed."
- **Big day:** rank events by house-severity, cap the output (top N events, top M
  updates each) with a "+X more" overflow, headline event first — so the one
  morning that matters stays readable.

## Prescribed internals (from blindspots, not up for debate)

- **House severity scale.** GDACS colors, PAGER colors, and ReliefWeb (no
  severity) are NOT the same measure. Pick one internal "house scale" and
  document the mapping from each feed as explicit policy — a naive unified
  "severity" column silently lies.
- **Fuzzy cross-feed entity resolution.** Joins are one-to-many, not one-to-one
  (two quakes → two GDACS events → one ReliefWeb disaster). USGS/GDACS IDs never
  match and GLIDE arrives days late, so the practical join is fuzzy: hazard type +
  time window + geographic distance, upgraded to GLIDE when it appears.
- **Dedup within ReliefWeb** (same sitrep from multiple sources) before cross-feed
  matching.

## Constraints & anchors

- Unattended, scheduled, publishes to `dashboard.html` at 08:30 SGT.
- Single deliverable repo following the template layout (`scripts/`, `skills/`,
  `feeds/`, `docs/`, `implementation-notes.md`).
- Persistent event store + raw-payload archive required from the first slice.
- Course artefacts expected: `prd.html`, `system-view.html`,
  `implementation-notes.md`, `dashboard.html`, `goal.md`, ≥1 skill.

## Explicitly out of scope

- All-hazard coverage (conflict, epidemics, industrial accidents, heatwaves).
- Real tsunami warnings (`tsunami:1` is only a NOAA-zone flag; PTWC/tsunami.gov
  are separate sources, not in scope).
- Global coverage outside Asia-Pacific.
- Sub-Orange / sub-PAGER-yellow event reporting (except brand-new-event handling
  is governed by the severity floor as written).

## Success criteria

- At 08:30 SGT the report reflects everything report-worthy since the last good
  report, with correct UTC-based timing and no silent 8-hour timestamp shift.
- It stays quiet (heartbeat only) on a genuinely quiet morning and does not spam
  on Green churn or Nth sitreps.
- It issues corrections when a headlined event is downgraded or deleted.
- The worst-day fixture replays cleanly and the demo is deterministic.
- ReliefWeb stays under its daily call budget.

## Open questions (for grill-with-docs to resolve)

- Exact house-severity scale definition and the per-feed mapping.
- Polling intervals per feed and the ReliefWeb per-run call budget number.
- "Material revision" thresholds (magnitude delta, impact delta).
- Report caps (N events, M updates) and the geographic bounding of "Asia-Pacific."
- Where the persistent store and raw archive live, and their retention.
