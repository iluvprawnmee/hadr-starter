# HADR Monitor

The shared language of a monitoring agent that watches three disaster feeds and
publishes a morning situation report for the Asia-Pacific region. This file is a
glossary only — it defines what terms mean, never how they are implemented.

## Domain — what we monitor

**Hazard**:
A natural threat event we track: earthquake, cyclone, flood, volcano, drought, or
wildfire. Conflict, epidemics, and industrial accidents are explicitly not hazards
in this context.
_Avoid_: disaster (reserved for ReliefWeb's declared record), incident.

**Event**:
A single occurrence of a hazard as reported by a detection feed — one earthquake,
one cyclone. Identified per feed by that feed's own key.
_Avoid_: alert, incident.

**Episode**:
A time-stamped update to an ongoing Event — a cyclone's revised forecast track, a
flood's re-issue. One Event has many Episodes over its life.
_Avoid_: revision (which means a corrected value, not a new update).

**Situation**:
The unit the report reasons about: an aggregate of one or more Events (across
feeds) plus any matched ReliefWeb disaster that all describe the same real-world
crisis. The join is many-to-one — two earthquakes can belong to one Situation.
_Avoid_: case, group, cluster.

**Disaster**:
ReliefWeb's curated, human-declared record that a crisis matters to the
international humanitarian system — carries official naming and a GLIDE. Slower and
sparser than detection Events.
_Avoid_: using "disaster" for a raw detection Event.

## Domain — severity & attention

**House severity scale**:
This project's single canonical severity ladder — **Notable**, **Serious**,
**Critical** — onto which each feed's own signal is mapped by policy. Feed scales
are never compared directly.
_Avoid_: level, priority, unified severity.

**Severity floor**:
The threshold below which an Event is never reported: GDACS Orange+, USGS PAGER
yellow+, or any ReliefWeb-declared disaster in region. Everything below is noise.
_Avoid_: cutoff, minimum.

**Alert level**:
A feed's own colour-coded impact signal (GDACS Green/Orange/Red; USGS PAGER
green/yellow/orange/red). Feed-native — mapped into the house scale, not used
across feeds directly.
_Avoid_: severity (reserved for the house scale), score.

**GLIDE**:
The global identifier a disaster receives once the humanitarian system names it
(e.g. `EQ-2026-000093-VEN`). Arrives days late; when present it firmly ties feeds
to one Situation.
_Avoid_: disaster id, event id.

## Domain — what counts as change

**News** (report-worthy change):
A change the morning report is allowed to surface. Only these qualify: a new
in-scope Event, an escalation, a material revision, a correction, the first
ReliefWeb signal on a known disaster (a disaster declaration now, a sitrep once the
API path is live), or a new Episode for a fast-moving hazard.
_Avoid_: update, change (too broad), alert.

**Escalation**:
An Event's alert level moving upward past a boundary (e.g. Orange → Red). Always
News.
_Avoid_: upgrade.

**Material revision**:
A corrected value large enough to matter — a magnitude change past threshold, or
any alert-level colour change. Small drift that changes no colour is not material.
_Avoid_: update, edit.

**Correction**:
The report retracting or downgrading something it previously headlined, because the
underlying Event was downgraded or deleted from its feed. A credible report issues
corrections.
_Avoid_: retraction, deletion.

**Fast-moving hazard**:
A hazard whose Episodes are themselves News: cyclone, flood, volcano. Slow-onset
hazards (drought) and rarely-updated ones (earthquake) are not fast-moving.
_Avoid_: dynamic, active.

**Lifecycle status**:
The tracked state of an Event as we understand it — `active`, `downgraded`, or
`deleted`. Deletion is inferred when an Event is not seen for several polls while
still inside the report window; it drives corrections, not silent removal.
_Avoid_: state, status (unqualified), stage.

## Domain — the report

**Situation report** (sitrep):
The published morning briefing — the agent's single external deliverable. Ranked
Situations, a corrections section, and a health footer.
_Avoid_: report (ambiguous), summary, digest.

**Report window**:
The span of time a given sitrep covers: everything since the last successfully
published report, defined in UTC. Never a fixed clock-day.
_Avoid_: period, range.

**Data-as-of**:
The moment a feed's data was last successfully retrieved, stamped per feed on the
report. Distinct from the report window.
_Avoid_: last updated, timestamp.

**Quiet day**:
A morning on which nothing meets the news threshold. The report still publishes —
a "no significant changes" note plus the health footer — so quiet is visibly
distinct from a crash.
_Avoid_: empty report, silence.

**Heartbeat / health footer**:
The always-present part of every report stating each feed is healthy and its
data-as-of time. Proves the agent ran.
_Avoid_: status bar, ping.

**Stale-data banner**:
A warning shown when a feed's data-as-of is too old (a feed was down at report
time). The report publishes anyway, flagged.
_Avoid_: error, warning (too generic).

## Domain — grounding & test

**Entity resolution**:
Deciding that Events and disasters from different feeds describe the same
real-world crisis, and grouping them into one Situation. Fuzzy by nature: hazard
type + time proximity + geographic proximity, firmed up by a shared GLIDE.
_Avoid_: matching, dedup (dedup is the narrower within-feed case).

**Fixture**:
A recorded real feed payload, frozen for deterministic replay and testing — the
agent is tested against fixtures, never live feeds.
_Avoid_: mock, sample, snapshot.

**Worst-day fixture**:
A hand-crafted fixture combining the nasty cases — a magnitude revision, an event
deletion, cyclone episode churn, and a feed-down — used to prove the diff and
correction logic.
_Avoid_: edge case, stress test.

**Replay**:
Feeding recorded fixtures through the pipeline as if they were live, to reproduce a
"real day" deterministically (including for the demo).
_Avoid_: simulation, playback.
