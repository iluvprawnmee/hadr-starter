# PRD — HADR Monitor

Synthesized from `REQS.md`, `CONTEXT.md`, `QUESTIONS.md`, and `docs/adr/0001–0008`.
Uses the project glossary throughout; respects all eight ADRs. Tuning values that
were left to the PRD in Step A are pinned here.

## Problem Statement

A humanitarian duty analyst anchored to Singapore time needs, every morning at
08:30, a trustworthy picture of which natural-hazard **Events** in Asia-Pacific are
genuinely new or changed since the last briefing. The raw feeds make this
impossible by hand: GDACS Green noise is ~95% of volume, USGS `all_day` is
dominated by tiny instrument-dense quakes, and ReliefWeb posts hundreds of items a
day. Worse, the feeds are rolling windows with no history and no deletion
tombstones, three incompatible time formats, and impact signals (PAGER, GDACS
colour) that arrive late and get revised. The analyst cannot tell "quiet" from
"crashed," cannot trust a report built on mis-parsed timestamps, and has no way to
know when yesterday's headline event was quietly downgraded or deleted.

## Solution

An unattended agent that continuously polls GDACS, USGS, and ReliefWeb into its own
persistent event store and raw-payload archive, normalizes all timestamps to UTC at
ingest, resolves cross-feed **Situations**, and applies an explicit editorial
change-detection policy. At 08:30 SGT it publishes a ranked **situation report** to
`dashboard.html` covering everything report-worthy since the last successful
report. On a **quiet day** it still publishes — a "no significant changes" note
plus a **health footer** — so quiet is visibly distinct from a crash. When a feed
is down it publishes anyway with a **stale-data banner**. When a headlined Event is
downgraded or deleted it issues a **correction**. On a big day it ranks by the
**house severity scale** and caps the output so the one morning that matters stays
readable.

## User Stories

**Core analyst — the morning report**

1. As a duty analyst, I want a single ranked situation report at 08:30 SGT, so that I can brief without reading raw feeds.
2. As a duty analyst, I want only report-worthy changes since the last report, so that I am not re-shown yesterday's news.
3. As a duty analyst, I want the headline Situation first, so that the most important crisis is unmissable.
4. As a duty analyst, I want each Situation to show what happened, where, how bad (house severity), and who is affected, so that I can assess it at a glance.
5. As a duty analyst, I want two related earthquakes and their ReliefWeb disaster shown as one Situation, so that I read one story, not three.
6. As a duty analyst, I want a "Corrections since last report" section, so that I learn when a prior headline was downgraded or deleted.
7. As a duty analyst, I want new cyclone forecast tracks and flood re-issues surfaced as news, so that I follow fast-moving hazards as they evolve.
8. As a duty analyst, I want static re-issues and Nth signals suppressed, so that ongoing events do not spam the report.

**Trust, quiet, and failure**

9. As a duty analyst, I want a "no significant changes" report on a quiet morning, so that I know the agent ran and nothing needs attention.
10. As a duty analyst, I want a per-feed health footer with each feed's data-as-of time, so that I trust the report's freshness.
11. As a duty analyst, I want a stale-data banner when a feed was down, so that I know the picture is partial rather than complete.
12. As a duty analyst, I want the report to still publish when one feed is down, so that a single outage does not blind me.
13. As a duty analyst, I want a genuinely dangerous fresh quake surfaced even before PAGER scores it, so that a large shallow event under a city is never missed.

**Scope and severity**

14. As a duty analyst, I want only Asia-Pacific natural hazards, so that the report is relevant to my region.
15. As a duty analyst, I want offshore and multi-country events scoped by coordinates, so that events with a missing or wrong country field are still placed correctly.
16. As a duty analyst, I want a single consistent severity ladder across feeds, so that I am not misled by GDACS and PAGER colours that look alike but mean different things.
17. As a duty analyst, I want sub-floor noise (GDACS Green, tiny quakes) excluded, so that the report stays signal.

**Operator / maintainer**

18. As the operator, I want the agent to run unattended on a schedule, so that no one has to trigger it.
19. As the operator, I want every raw payload archived before parsing, so that schema drift breaks parsing, not history.
20. As the operator, I want a persistent event store from the first slice, so that change, deletion, and corrections can be detected at all.
21. As the operator, I want ReliefWeb calls budgeted per run, so that I stay under the 1,000/day cap.
22. As the operator, I want feed errors to back off gracefully and never fail the whole run, so that one flaky feed degrades to a banner.
23. As the operator, I want a missed scheduled run to be absorbed by the "since last successful report" window, so that no events are dropped.

**Demo / reviewer (course)**

24. As a reviewer, I want a deterministic replay of a recorded "real day," so that the demo does not depend on live feeds being interesting.
25. As a reviewer, I want a synthetic "worst-day" fixture exercising revision, deletion, cyclone churn, and a feed-down, so that the diff and correction logic is proven on the nasty cases.
26. As a reviewer, I want tests that run against fixtures, not live feeds, so that the suite is deterministic and offline.
27. As a reviewer, I want implementation decisions and deviations logged, so that I can review work I did not write.

## Vertical Slices

The system is built outside-in as three independently demo-able slices, each ending
in a real `dashboard.html`. This mirrors `prd.html` and the selected Shape C
(per-entity tracker) in `SHAPING.md`.

**V1 — Skeleton report, one feed** (ingest → store → render, end to end)
- Build: GDACS adapter → raw-payload archive → normalizer (UTC + house severity) →
  SQLite store → Asia-Pacific bounding-box & severity-floor filter → ranker → HTML
  renderer.
- Demo: this morning's in-region GDACS Serious+ events as a ranked report; first run
  covers the whole window (no diffing yet).
- Proves: R0, R1.1, R1.2, R4.1, R5.1, R5.2, R7.

**V2 — The diff engine & Situations, all three feeds** (the heart: decide what is
news)
- Build: add USGS + ReliefWeb adapters; entity tracker + fuzzy resolver grouping
  feed records into Situations; transition engine emitting News (new, escalation,
  material revision, new episode); diff against the last report; quiet-day note +
  per-feed health footer.
- Demo: replay a recorded two-run day from fixtures — the second report shows only
  what changed; a quiet run shows just the heartbeat.
- Proves: R1.3, R2.1, R2.2, R2.3, R3.1, R4.2, R6.1, R8.1, R8.3.

**V3 — Corrections, resilience & the schedule** (trustworthy + unattended)
- Build: per-entity downgrade / "not-seen-for-K-polls → deleted" transitions feeding
  a Corrections section; stale-data banner + graceful feed-down; ReliefWeb call
  budget & backoff; GitHub Actions cron at 00:30 UTC committing `dashboard.html`;
  synthetic worst-day fixture.
- Demo: the worst-day fixture yields a Critical event, a correction, and a stale
  banner; the scheduled workflow runs unattended and commits the report.
- Proves: R3.2, R3.3, R6.2, R6.3, R8.2, scheduling.

## Implementation Decisions

Language, scheduler, persistence, ReliefWeb source, house scale, entity
resolution, geographic scope, and change semantics are fixed by `ADR-0001`–`0008`
respectively and are not re-litigated here. This section pins the module shape and
the tuning values.

**Pipeline modules (single-responsibility units)**

- **Feed adapters** — one per source (GDACS GeoJSON, USGS GeoJSON, ReliefWeb RSS),
  each exposing the same interface: fetch → return raw payload. ReliefWeb sits
  behind an adapter so the `reports` API can replace RSS later (`ADR-0004`).
- **Archiver** — writes every raw payload to a timestamped file before parsing
  (`ADR-0003`).
- **Normalizer** — parses each feed's format and produces feed records with
  timezone-aware UTC times (GDACS naive→UTC, USGS epoch-ms, ReliefWeb RFC-822) and
  a house-severity level via the `ADR-0005` mapping.
- **Store** — the persistent normalized event store; upserts feed records keyed by
  GDACS `(eventtype, eventid)`, USGS `id`, ReliefWeb GLIDE (fallback `link`).
- **Entity resolver** — groups feed records into **Situations** by fuzzy match
  (`ADR-0006`).
- **Change detector** — a pure function: (previous report state, current store
  state) → the list of News items (`ADR-0008`). This is the heart of the system.
- **Ranker** — orders Situations by house severity (max across matched records),
  applies caps.
- **Renderer** — emits `dashboard.html` from ranked Situations + corrections +
  health footer.
- **Scheduler** — two GitHub Actions crons (`ADR-0002`): a frequent poll workflow
  feeding the store, and a report workflow at 00:30 UTC that renders and commits
  `dashboard.html`. Both persist the store via the `data` branch (`ADR-0009`).

**Severity floor + house mapping** (`ADR-0005`, `ADR-0008`): report-worthy =
GDACS Orange+ / PAGER yellow+ / any ReliefWeb declaration in region, with exactly
one exception — an in-region USGS quake with `alert = null` is admitted when
**M ≥ 6.5 and depth ≤ 70 km** (PAGER-lag fallback), reported provisionally and
reconciled once PAGER populates.

| Source signal | House level |
| --- | --- |
| GDACS Orange | Serious |
| GDACS Red | Critical |
| PAGER yellow | Notable |
| PAGER orange | Serious |
| PAGER red | Critical |
| ReliefWeb declaration (no severity) | Notable |

**Change semantics** (`ADR-0008`): News = new Event · escalation · material
revision (USGS |Δmag| ≥ 0.5, or any GDACS alert-colour change) · correction · first
ReliefWeb signal (declaration/new GLIDE in the MVP) · new Episode for a fast-moving
hazard (cyclone/flood/volcano). GDACS event-level `alertlevel` drives the floor and
headline; episode level drives fast-moving-hazard episode news. Corrections remain
eligible for 7 days after last headline.

**Entity resolution** (`ADR-0006`): same hazard type, origin time within ±24h,
epicentre/centroid within ~250 km; promote to a firm match on shared GLIDE.
Within-ReliefWeb duplication is deduped before cross-feed matching.

**Geographic scope** (`ADR-0007`): bounding box ≈ lon 60°E–180°, lat 45°N–50°S;
coordinates authoritative over the `country` field; hazard types EQ, TC, FL, VO,
DR, WF.

**Report window & timing** (`ADR-0002`, `ADR-0008`): window = since last
successful report, UTC; first-ever run defaults to the last 24h. Data-as-of stamped
per feed. Stale-data banner when a feed's last successful fetch is older than
**2× its polling interval or 90 min, whichever is greater**.

**Polling cadence** (`ADR-0004`): GDACS 15 min, USGS 15 min, ReliefWeb 60 min;
ReliefWeb calls budgeted per run; exponential backoff on error with last-good
retained.

**Dashboard layout & caps**: headline Situation → ranked Situation cards →
"Corrections since last report" section → health/heartbeat footer. Big-day caps:
**top 8 Situations, top 5 updates each, "+X more" overflow**. Quiet-day body:
"No significant changes since {last report time}." + footer.

## Data Model

A per-entity tracker in SQLite (the persistent memory the feeds lack, `ADR-0003`),
plus the raw archive written before parsing. Both persist between ephemeral CI runs
on a dedicated `data` branch (`ADR-0009`). Severity is stored on the house scale
(Notable/Serious/Critical, `ADR-0005`), never a feed's native colour.

| Table | Represents | Key fields |
| --- | --- | --- |
| `raw_payload` | Every fetch, archived before parsing (schema-drift + fixture source) | `id, feed, fetched_at(UTC), http_status, body` |
| `event` | One tracked hazard Event per feed record (a lifecycle entity) | `id, feed, feed_key, hazard_type, occurred_at(UTC), lat, lon, depth, magnitude, feed_alert, house_severity, country, iso3, glide, first_seen, last_seen_poll, lifecycle_status, raw_ref` |
| `episode` | Updates to a fast-moving Event (cyclone track, flood re-issue) | `id, event_id→, episode_key, updated_at(UTC), alert_level` |
| `situation` | Cross-feed aggregate — the unit the report tells as one story | `id, glide, hazard_type, house_severity(max), centroid_lat, centroid_lon, window_start` |
| `situation_member` | Many-to-one link: several Events → one Situation | `situation_id→, event_id→, role` |
| `report` | Each published morning report | `id, published_at(UTC), window_start, window_end` |
| `news_item` | What appeared in a report, and why | `id, report_id→, situation_id→, news_type, house_severity, rank` |
| `feed_health` | Per-feed freshness driving the stale-data banner | `feed, last_success_at(UTC), last_status` |

**Relationships:** `event *→1 situation` (via `situation_member`); `situation 1→*
news_item` across reports; `event 1→* episode`; every `event` and parse traces back
to a `raw_payload`.

**Enumerations:** `news_type ∈ {new, escalation, revision, episode, correction}`;
`lifecycle_status ∈ {active, downgraded, deleted}`.

## Testing Decisions

- **What makes a good test here:** it asserts on **external behavior** — the
  published `dashboard.html` and the store state — never on internal function
  calls. The pipeline is deterministic given a fixed set of input payloads.
- **Primary seam — fixture replay at the feed-adapter boundary.** Recorded raw
  payloads are fed through the real pipeline (archiver → normalizer → store →
  resolver → change detector → renderer) and the resulting `dashboard.html` +
  store state are asserted. This is the single highest seam; live feeds are never
  contacted in tests (`ADR-0003`, REQS Q6).
- **Secondary seam — the change detector as a pure function.** (prev report state,
  current store) → News list. This isolates the hardest logic — escalation,
  material revision, corrections, fast-moving episodes, the PAGER-lag fallback —
  for direct, fast assertion.
- **Modules tested:** normalizer (three time formats → UTC; house mapping),
  entity resolver (one-to-many, GLIDE promotion, offshore/empty-country), change
  detector (all News rules + suppression of static re-issues), renderer (quiet day,
  big-day caps, corrections section, stale banner).
- **Signature fixtures:** a recorded "real day" including a Red event (demo replay),
  and the synthetic **worst-day fixture** (magnitude revision, event deletion,
  cyclone episode churn, one feed down) proving the diff and correction paths.
- **Prior art:** deterministic checks live in `scripts/` per the template; the
  `docs/solutions/` follow-redirects example is the model for capturing a learning.

## Out of Scope

- All-hazard coverage — conflict, epidemics, industrial accidents, heatwaves,
  landslides (the three feeds cannot deliver them).
- Real tsunami warnings (`tsunami:1` is only a NOAA-zone flag; PTWC/tsunami.gov are
  separate sources).
- Global coverage outside the Asia-Pacific bounding box.
- Sub-floor reporting (GDACS Green, PAGER green/none, tiny quakes) except the single
  M≥6.5/depth≤70km PAGER-lag exception.
- ReliefWeb casualty figures and sitrep counts — blocked on the appname-gated
  `reports` API; the MVP surfaces disaster declarations / GLIDE only (`ADR-0004`).
- Historical backfill beyond what the agent's own store accumulates from first run.

## Further Notes

- **Appname risk:** ReliefWeb `reports` API access may not arrive during the build
  week; the RSS-first adapter (`ADR-0004`) keeps the MVP unblocked and the upgrade
  path clean.
- **PAGER & GDACS revision:** impact signals arrive late and change; the store +
  correction policy exist precisely so early polls and later revisions reconcile
  rather than lie.
- **Schema drift:** GDACS's schema is semi-documented and shifts without notice —
  the raw archive is kept even when parsing breaks.
- **Big-day = quiet-day's evil twin:** a real Red produces GDACS episode churn plus
  a flood of ReliefWeb items; caps and ranking are what keep the pipeline and the
  report from choking on the morning that matters most.
- **Single point of failure:** for every hazard except earthquakes, GDACS is the
  sole detection feed — worth watching in operation.
