---
shaping: true
---

# HADR Monitor — Shaping

Working document for shaping the HADR Monitor. Ground truth for requirements (R),
shapes, fit checks, and the detailed shape. See `FRAME.md` for the why,
`docs/PRD.md` for user stories, `CONTEXT.md` for vocabulary, `docs/adr/` for fixed
decisions.

## Requirements (R)

Top-level requirements are capped at 9; detail lives in sub-requirements.

| ID | Requirement | Status |
|----|-------------|--------|
| **R0** | Publish a ranked situation report to `dashboard.html` at 08:30 SGT covering everything report-worthy since the last successful report | Core goal |
| **R1** | **Report only report-worthy Asia-Pacific natural-hazard signal — filter the noise** | Must-have |
| R1.1 | Scope events to Asia-Pacific by coordinate bounding box, coordinates authoritative over the country field | Must-have |
| R1.2 | Apply the severity floor (GDACS Orange+ / PAGER yellow+ / ReliefWeb declaration), with the M≥6.5 / depth≤70km PAGER-lag exception | Must-have |
| R1.3 | Re-surface a known Situation only on News (new event, escalation, material revision, correction, first ReliefWeb signal, fast-moving-hazard episode); suppress static re-issues and Nth signals | Must-have |
| **R2** | **Present one story per real-world crisis, not one per feed record** | Must-have |
| R2.1 | Resolve Events/disasters across feeds into Situations by fuzzy match (hazard + ±24h + ~250km), promoted on shared GLIDE | Must-have |
| R2.2 | Deduplicate within ReliefWeb before cross-feed matching | Must-have |
| R2.3 | Represent one-to-many crises (several Events → one Situation) as a Situation aggregate | Must-have |
| **R3** | **Be honest about the report's own state** | Must-have |
| R3.1 | On a quiet day still publish — a "no significant changes" note plus the health footer (quiet ≠ silent) | Must-have |
| R3.2 | When a feed is down, still publish with a per-feed stale-data banner | Must-have |
| R3.3 | Issue corrections when a headlined Event is downgraded or deleted (eligible 7 days after last headline) | Must-have |
| **R4** | **Never lie about timing; never silently drop a dangerous event** | Must-have |
| R4.1 | Normalize all three feed time formats to timezone-aware UTC at ingest | Must-have |
| R4.2 | Define the report window as "since last successful report" in UTC; first-ever run = last 24h | Must-have |
| **R5** | **Keep persistent history the feeds do not** | Must-have |
| R5.1 | Maintain a persistent normalized event store from the first slice | Must-have |
| R5.2 | Archive every raw payload before parsing | Must-have |
| **R6** | **Run unattended and degrade gracefully** | Must-have |
| R6.1 | Run on a schedule and absorb a missed run without dropping events | Must-have |
| R6.2 | Back off on feed errors and never fail the whole run over one feed | Must-have |
| R6.3 | Budget ReliefWeb calls per run to stay under 1,000/day | Must-have |
| **R7** | Stay readable on a big day — rank Situations by house severity and cap output (top 8 events, top 5 updates each, "+X more") | Must-have |
| **R8** | **Be deterministically replayable and testable** | Must-have |
| R8.1 | Replay a recorded "real day" (incl. a Red event) deterministically for the demo | Must-have |
| R8.2 | Exercise a synthetic worst-day fixture (revision, deletion, cyclone churn, feed-down) | Nice-to-have |
| R8.3 | Test against fixtures, never live feeds | Must-have |

**Status vocabulary:** Core goal · Must-have · Nice-to-have · Undecided · Out

---

## Shapes (S)

Three mutually exclusive approaches. They share ingest/render/schedule
infrastructure (fixed by the ADRs); they differ in how state is stored and how
**change is detected** — the diff engine. Flags (⚠️) mark mechanisms we can name
but do not yet know concretely how to build.

### A: Snapshot-diff batch pipeline

Detect change by comparing the current resolved state against the state captured in
the last report.

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **A1** | Feed adapters (GDACS/USGS/ReliefWeb) — fetch raw payload behind a common interface | |
| **A2** | Archiver — write each raw payload to a timestamped file before parsing | |
| **A3** | Normalizer — parse each format → UTC feed records + house-severity mapping | |
| **A4** | Current-state store (SQLite) — upsert latest normalized record per feed key | |
| **A5** | Entity resolver — group current records into Situations by fuzzy match | ⚠️ |
| **A6** | Report snapshot — persist the set of Situations shown in each report | |
| **A7** | Snapshot differ — compare current Situations vs last snapshot → new / escalation / material revision / new episode | |
| **A8** | Absence handler — a Situation in the last snapshot but missing now → correction | ⚠️ |
| **A9** | Ranker + caps (top 8 / top 5, overflow) | |
| **A10** | Renderer → `dashboard.html` (headline, cards, corrections, health footer) | |
| **A11** | Scheduler (GitHub Actions cron 00:30 UTC) + polling loop | |

### B: Event-sourced observation log

Append every observation to an immutable log; project state and classify News by
querying the log since the last report.

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **B1** | Feed adapters | |
| **B2** | Archiver | |
| **B3** | Observation recorder — normalize + append one immutable observation per feed record per poll (UTC obs-time) | |
| **B4** | State projector — fold the observation log into current Events + Situations | |
| **B5** | Entity resolver (within projection) — fuzzy match | ⚠️ |
| **B6** | News classifier — query observations since last report; classify transitions, incl. prolonged-absence → deletion | ⚠️ |
| **B7** | Ranker + caps | |
| **B8** | Renderer → `dashboard.html` | |
| **B9** | Scheduler + polling loop | |

### C: Per-entity state-machine tracker

Track each Event/Situation as an entity with an explicit lifecycle; emit News when
a transition fires.

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **C1** | Feed adapters | |
| **C2** | Archiver | |
| **C3** | Normalizer → UTC records + house-severity | |
| **C4** | Entity tracker store — one row per tracked Event/Situation: current fields, lifecycle status, last-seen poll | |
| **C5** | Entity resolver — attach each incoming record to an existing entity or create one (fuzzy) | ⚠️ |
| **C6** | Transition engine — per poll, compute per-entity transitions (new / escalate / revise / new-episode / downgrade / not-seen-for-K-polls → deleted) and emit News to a pending queue | |
| **C7** | Report builder — drain the pending-News queue since last report, rank + cap | |
| **C8** | Renderer → `dashboard.html` | |
| **C9** | Scheduler + polling loop | |

---

## Fit Check v1

| Req | Requirement | Status | A | B | C |
|-----|-------------|--------|:-:|:-:|:-:|
| R0 | Ranked report to `dashboard.html` at 08:30 SGT, everything report-worthy since last report | Core goal | ✅ | ✅ | ✅ |
| R1.1 | Scope events to Asia-Pacific by coordinate bounding box, coordinates authoritative | Must-have | ✅ | ✅ | ✅ |
| R1.2 | Severity floor + M≥6.5/≤70km PAGER-lag exception | Must-have | ✅ | ✅ | ✅ |
| R1.3 | Re-surface only on News; suppress static re-issues / Nth signals | Must-have | ✅ | ✅ | ✅ |
| R2.1 | Fuzzy cross-feed resolution → Situations, GLIDE upgrade | Must-have | ❌ | ❌ | ❌ |
| R2.2 | Deduplicate within ReliefWeb before cross-feed matching | Must-have | ✅ | ✅ | ✅ |
| R2.3 | Represent one-to-many crises as a Situation aggregate | Must-have | ✅ | ✅ | ✅ |
| R3.1 | Quiet day still publishes + health footer | Must-have | ✅ | ✅ | ✅ |
| R3.2 | Feed-down still publishes + stale banner | Must-have | ✅ | ✅ | ✅ |
| R3.3 | Corrections on downgrade/deletion (7-day window) | Must-have | ❌ | ❌ | ✅ |
| R4.1 | Normalize all feed times to UTC at ingest | Must-have | ✅ | ✅ | ✅ |
| R4.2 | Window = since last good report; first run = 24h | Must-have | ✅ | ✅ | ✅ |
| R5.1 | Persistent normalized event store from slice 1 | Must-have | ✅ | ✅ | ✅ |
| R5.2 | Archive every raw payload before parsing | Must-have | ✅ | ✅ | ✅ |
| R6.1 | Scheduled; absorbs a missed run without dropping events | Must-have | ✅ | ✅ | ✅ |
| R6.2 | Feed-error backoff; never fail whole run | Must-have | ✅ | ✅ | ✅ |
| R6.3 | Budget ReliefWeb calls under 1,000/day | Must-have | ✅ | ✅ | ✅ |
| R7 | Rank by house severity + caps | Must-have | ✅ | ✅ | ✅ |
| R8.1 | Replay recorded "real day" deterministically | Must-have | ✅ | ✅ | ✅ |
| R8.2 | Synthetic worst-day fixture | Nice-to-have | ✅ | ✅ | ✅ |
| R8.3 | Test against fixtures, never live | Must-have | ✅ | ✅ | ✅ |

**Notes:**
- All three fail **R2.1**: fuzzy cross-feed entity resolution is flagged (A5/B5/C5) — we can name it but not yet build it. This is a **shared unknown**: it needs a spike regardless of which shape wins, so it does not discriminate between shapes.
- A and B fail **R3.3**: both detect a downgrade fine, but a *deletion* (USGS records vanish with no tombstone; GDACS `istemporary` events disappear) reaches them only as *absence* — indistinguishable from a rolling-window age-out. A8/B6 are flagged for exactly this.
- C passes **R3.3**: the entity tracker keeps a `last-seen` poll and an explicit "not seen for K polls while still in window → deleted" transition (C6), which concretely separates deletion from age-out. This is the discriminator.
- Everything else passes for all three: the ingest, timing, persistence, operations, ranking, and replay mechanisms are the same across shapes and concretely understood.

## Fit Check v2

Re-run after resolving R2.1 via [`SPIKE-entity-resolution.md`](./SPIKE-entity-resolution.md),
which gives a concrete per-hazard matching mechanism. Only R2.1 changes (🟡).

| Req | Requirement | Status | A | B | C |
|-----|-------------|--------|:-:|:-:|:-:|
| R0 | Ranked report to `dashboard.html` at 08:30 SGT, everything report-worthy since last report | Core goal | ✅ | ✅ | ✅ |
| R1.1 | Scope events to Asia-Pacific by coordinate bounding box, coordinates authoritative | Must-have | ✅ | ✅ | ✅ |
| R1.2 | Severity floor + M≥6.5/≤70km PAGER-lag exception | Must-have | ✅ | ✅ | ✅ |
| R1.3 | Re-surface only on News; suppress static re-issues / Nth signals | Must-have | ✅ | ✅ | ✅ |
| R2.1 | 🟡 Fuzzy cross-feed resolution → Situations, GLIDE upgrade | Must-have | ✅ | ✅ | ✅ |
| R2.2 | Deduplicate within ReliefWeb before cross-feed matching | Must-have | ✅ | ✅ | ✅ |
| R2.3 | Represent one-to-many crises as a Situation aggregate | Must-have | ✅ | ✅ | ✅ |
| R3.1 | Quiet day still publishes + health footer | Must-have | ✅ | ✅ | ✅ |
| R3.2 | Feed-down still publishes + stale banner | Must-have | ✅ | ✅ | ✅ |
| R3.3 | Corrections on downgrade/deletion (7-day window) | Must-have | ❌ | ❌ | ✅ |
| R4.1 | Normalize all feed times to UTC at ingest | Must-have | ✅ | ✅ | ✅ |
| R4.2 | Window = since last good report; first run = 24h | Must-have | ✅ | ✅ | ✅ |
| R5.1 | Persistent normalized event store from slice 1 | Must-have | ✅ | ✅ | ✅ |
| R5.2 | Archive every raw payload before parsing | Must-have | ✅ | ✅ | ✅ |
| R6.1 | Scheduled; absorbs a missed run without dropping events | Must-have | ✅ | ✅ | ✅ |
| R6.2 | Feed-error backoff; never fail whole run | Must-have | ✅ | ✅ | ✅ |
| R6.3 | Budget ReliefWeb calls under 1,000/day | Must-have | ✅ | ✅ | ✅ |
| R7 | Rank by house severity + caps | Must-have | ✅ | ✅ | ✅ |
| R8.1 | Replay recorded "real day" deterministically | Must-have | ✅ | ✅ | ✅ |
| R8.2 | Synthetic worst-day fixture | Nice-to-have | ✅ | ✅ | ✅ |
| R8.3 | Test against fixtures, never live | Must-have | ✅ | ✅ | ✅ |

**Notes:**
- R2.1 now passes for all shapes — the matching mechanism is shape-independent.
- **R3.3 remains the sole discriminator**: only Shape C concretely separates a
  deletion (no tombstone) from a rolling-window age-out, via the entity tracker's
  `last-seen` + "not seen for K polls → deleted" transition. **Shape C passes every
  requirement; A and B do not.**

## Spikes

- [`SPIKE-entity-resolution.md`](./SPIKE-entity-resolution.md) — **done.** Resolved
  R2.1 with per-hazard tolerances (EQ ±30min/150km, TC ±24h/500km, RW GLIDE-or-
  country/±72h…), a blocking + GLIDE-first + fuzzy-score algorithm, and the
  one-to-many Situation aggregation. The ⚠️ on the resolver part is cleared.
- **Surfaced during detailing, now resolved:** cross-run persistence on ephemeral
  CI → commit the store + rolling archive to a `data` branch (`ADR-0009`). C9 flag
  cleared.

## Selected shape

**Shape C — Per-entity state-machine tracker.** It is the only shape that passes the
full fit check (v2): it satisfies R3.3 because tracking each Event/Situation as a
lifecycle entity with a `last-seen` poll makes deletion an explicit, designed
transition rather than ambiguous absence — directly answering the blindspots.md
"rolling windows, not logs" trap. R2.1 is resolved and shape-independent, so it does
not count against A/B; R3.3 is decisive.

## Detail C — concrete components

Expansion of Shape C into buildable components (not an alternative). Each is a
vertical slice pairing mechanism with the data it owns. One flagged unknown remains
(C9), surfaced during detailing.

| Part | Mechanism | Flag |
|------|-----------|:----:|
| **C1** | **Feed adapters** — GDACS GeoJSON (`geteventlist/EVENTS4APP`), USGS `all_day.geojson`, ReliefWeb `disasters` RSS; common interface `fetch() → (raw_bytes, fetched_at, http_status)`. ReliefWeb behind the adapter so the `reports` API can replace RSS (`ADR-0004`). | |
| **C2** | **Archiver** — write each raw payload to `archive/{feed}/{fetched_at}.{ext}` and insert a `raw_payload` row, before any parsing (`ADR-0003`). | |
| **C3** | **Normalizer** — per-feed parser → normalized record: `hazard_type`, `occurred_at` UTC (GDACS naive→UTC, USGS epoch-ms, RW RFC-822), `lat/lon/depth/mag`, `feed_alert`, `house_severity` (`ADR-0005`), `country/iso3`, `glide`, `feed_key`. Applies AP bounding box (`ADR-0007`), severity floor + M≥6.5/≤70km fallback (`ADR-0008`). | |
| **C4** | **Entity tracker store** — SQLite per the PRD data model (`raw_payload · event · episode · situation · situation_member · report · news_item · feed_health`). Upsert each record into its `event` by `feed_key`; bump `last_seen_poll`. | |
| **C5** | **Entity resolver** — per `SPIKE-entity-resolution.md`: block by `hazard + geo-cell + time-bucket`; GLIDE-first join; per-hazard fuzzy score → attach/create Event; aggregate into Situation; ReliefWeb one-to-many attach; GLIDE upgrade; ReliefWeb dedup (R2.2). | |
| **C6** | **Transition engine** — per poll, compute per-entity transitions → News: `new` (first seen, above floor), `escalation` (house severity up), `revision` (\|Δmag\|≥0.5 or GDACS colour change), `episode` (TC/FL/VO episode change), `downgrade` + `deleted` (not seen for K polls while in-window). Emit `news_item` candidates. | |
| **C7** | **Report builder** — on the 08:30 run: window = since last successful report (first run 24h); gather News; rank Situations by house severity + recency; caps (top 8 / top 5, overflow); assemble Corrections; compute `feed_health` for the banner; quiet-day path when no News. | |
| **C8** | **Renderer** — deterministic `dashboard.html`: headline Situation → ranked cards → Corrections section → health/heartbeat footer, with stale banner when a feed is old. | |
| **C9** | **Scheduler + poller** — **two** GitHub Actions crons: a frequent poll-and-archive workflow feeding the store, and the 00:30 UTC report workflow that builds + renders + commits `dashboard.html` (`ADR-0002`); ReliefWeb budget + backoff (`ADR-0004`, `R6.3`). Cross-run persistence: both workflows read/write the SQLite store + rolling raw archive on a dedicated `data` branch (`ADR-0009`), so state survives ephemeral runners. | |

**All parts understood — no flags remain.** C9's cross-run persistence is resolved
by `ADR-0009` (commit store + rolling archive to a `data` branch, superseding the
gitignore consequence of `ADR-0003`). **Shape C is ready to breadboard (Step D).**
