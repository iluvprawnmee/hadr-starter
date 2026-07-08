# QUESTIONS — grilling backlog (Step A1)

Scratch file for `/grill-with-docs`. All open questions logged upfront so they can
be batch-answered; new ones appended as they surface. Each carries a **Rec**
(recommended answer). Mark `[x]` + record the decision when resolved; ADR-worthy
decisions get an `→ ADR` note.

Legend: `[ ]` open · `[x]` answered · `→ ADR` becomes an architectural decision record

> **STATUS (2026-07-08): all 25 questions answered — user accepted every
> recommendation as written.** ADR-worthy decisions captured in `docs/adr/`.
> See the resolved log at the bottom for the ADR cross-reference.

---

## A. Scope precision

- [ ] **A1. What bounds "Asia-Pacific"?** GDACS `country`/`iso3` can be empty
  (offshore quakes) or wrong (multi-country cyclones), so a name list alone
  fails.
  **Rec:** authoritative test is a lat/lon **bounding box** (approx lon 60°E–180°,
  lat 45°N–50°S); also admit any event whose `iso3` is in an AP country list.
  Coordinates win when country is empty/ambiguous. → ADR

- [ ] **A2. Include events with missing country if coordinates fall in-region?**
  **Rec:** yes — coordinates are authoritative over the `country` field.

- [ ] **A3. Which GDACS `eventtype`s count as in-scope natural hazards?**
  **Rec:** EQ (earthquake), TC (cyclone), FL (flood), VO (volcano), DR (drought),
  WF (wildfire) — all six. USGS supplies EQ redundantly.

## B. House severity scale

- [ ] **B1. Define the house scale levels.**
  **Rec:** three tiers — **Notable / Serious / Critical** — plus an implicit
  "below floor" that is never reported.
  → ADR (this is the "unified severity silently lies" trap)

- [ ] **B2. Per-feed mapping into the house scale.**
  **Rec:**
  | Source signal | House level |
  | --- | --- |
  | GDACS Orange | Serious |
  | GDACS Red | Critical |
  | PAGER yellow | Notable |
  | PAGER orange | Serious |
  | PAGER red | Critical |
  | ReliefWeb declared disaster (no severity) | Notable (until corroborated) |

- [ ] **B3. When an event has signals from several feeds, which house level wins?**
  **Rec:** the **maximum** house level across all matched feed records.

## C. Change semantics & thresholds

- [ ] **C1. GDACS: which field is "the alert level"?** (`alertlevel` vs
  `episodealertlevel`)
  **Rec:** **event-level `alertlevel`** governs the severity floor and headline;
  **episode-level** changes are tracked separately to drive cyclone/flood
  "new episode" news.

- [ ] **C2. "Material revision" — magnitude threshold (USGS).**
  **Rec:** report a magnitude revision when |Δmag| ≥ **0.5**, OR the revision
  crosses the severity floor, OR PAGER `alert` changes level.

- [ ] **C3. "Material revision" — GDACS impact threshold.**
  **Rec:** any `alertlevel` colour change is material; `alertscore` drift without
  a colour change is not.

- [ ] **C4. Which hazards are "fast-moving" (episode change = news)?**
  **Rec:** TC (cyclone), FL (flood), VO (volcano). NOT EQ (episodes rare) or DR
  (drought — slow, would spam).

- [ ] **C5. How long do we keep watching a headlined event for
  downgrade/deletion (corrections)?**
  **Rec:** while it remains within the report window and in the store's "active"
  set; retain correction-eligibility for **7 days** after last headline.

## D. Report window & time

- [ ] **D1. Confirm run cadence: 08:30 SGT = 00:30 UTC; data-as-of = last
  successful fetch per feed.**
  **Rec:** yes.

- [ ] **D2. First-ever run (no previous good report) — what window?**
  **Rec:** default to the **last 24h** on first run.

- [x] **D3. Stale-data banner threshold — how old before a feed is flagged?**
  **Resolved (A2):** flag when last successful fetch is older than **2× the polling
  interval OR 90 min, whichever is greater** — a per-feed grace floor so a single
  missed 15-min poll doesn't cry wolf.

## E. Cadence, budget & sources

- [ ] **E1. Polling interval per feed.**
  **Rec:** GDACS every 15 min, USGS every 15 min, ReliefWeb every 60 min.

- [ ] **E2. ReliefWeb: RSS or approved-appname API?** (appname approval may not
  arrive this week; API needs it, RSS does not)
  **Rec:** build against **RSS now** behind an adapter interface so the API can
  slot in later without touching the pipeline. → ADR

- [ ] **E3. ReliefWeb content type: `disasters` vs `reports`?** (blindspots: the
  substance lives in `reports`, but that needs the API)
  **Rec:** MVP uses **disasters RSS** for naming/GLIDE; upgrade to **reports API**
  for casualty/sitrep detail once appname clears. Record the gap as a known risk.
  → ADR

- [ ] **E4. Behaviour on feed error / rate-limit.**
  **Rec:** exponential backoff, keep last-good state in the store, never fail the
  whole run — a single feed down degrades gracefully to a banner.

## F. Entity resolution

- [ ] **F1. Fuzzy join parameters (hazard + time + geo).**
  **Rec:** same hazard type, origin time within **±24h**, epicentre/centroid
  within **~250 km**; promote to a firm match when a shared **GLIDE** appears.
  → ADR

- [ ] **F2. NEIC-sourced GDACS earthquakes duplicate USGS — how to dedupe?**
  **Rec:** treat GDACS EQ + USGS as the same physical event via F1; prefer USGS
  **PAGER** for impact, GDACS for cross-hazard consistency.

- [ ] **F3. One-to-many (two quakes → one ReliefWeb disaster) — data shape?**
  **Rec:** a **Situation** aggregate that holds one or more hazard Events plus any
  matched ReliefWeb disaster(s). → ADR

## G. Persistence & data policy

- [ ] **G1. Store technology + raw archive layout.**
  **Rec:** **SQLite** file for the normalized event store; raw payloads written as
  timestamped JSON/XML files in an archive dir, one per fetch. → ADR

- [ ] **G2. Retention & what's committed to git.**
  **Resolved (shaping C9 / `ADR-0009`):** the store + a rolling raw archive are
  committed to a dedicated **`data` branch** (the repo is the persistence layer,
  since CI runners are ephemeral); **fixtures** and `dashboard.html` live on `main`.

- [ ] **G3. Primary keys per feed.**
  **Rec:** GDACS `(eventtype, eventid)`; USGS `id`; ReliefWeb GLIDE (fallback to
  the `link` URL when GLIDE absent).

## H. Report / dashboard

- [ ] **H1. `dashboard.html` structure.**
  **Rec:** headline event → ranked event cards → "Corrections since last report"
  section → feed-health/heartbeat footer (per-feed last-fetch time).

- [ ] **H2. Big-day caps.**
  **Rec:** top **8** events, top **5** updates each, with "+X more" overflow.

- [ ] **H3. Quiet-day content.**
  **Rec:** "No significant changes since {last report time}." + the health footer,
  so quiet is visibly distinct from crashed.

- [ ] **H4. Corrections placement.**
  **Rec:** a dedicated "Corrections since last report" section above the footer.

## I. Tooling & operations (CLAUDE.md is currently empty)

- [ ] **I1. Language & runtime?** (CLAUDE.md has no language set)
  **Rec:** **Python 3.12** — stdlib `sqlite3`/`zoneinfo`, strong feed/date
  libraries; matches deterministic-scripts convention. → ADR

- [ ] **I2. Test command?**
  **Rec:** `pytest`, asserting against recorded fixtures (not live feeds).

- [ ] **I3. Scheduling mechanism for the 08:30 run?** (README mentions a
  `.github` sitrep workflow)
  **Rec:** **GitHub Actions** scheduled workflow at 00:30 UTC that runs the
  pipeline and commits `dashboard.html`. → ADR

- [ ] **I4. Timezone/date parsing approach?**
  **Rec:** normalize all three formats to timezone-aware UTC at ingest using
  stdlib `zoneinfo` + explicit parsers per feed (GDACS naive→UTC, USGS epoch-ms,
  ReliefWeb RFC-822).

---

## Resolved decisions log

All resolved 2026-07-08 — every recommendation accepted as written. ADR
cross-reference:

| Questions | Decision | ADR |
| --- | --- | --- |
| A1–A3 | Asia-Pacific by coordinate bounding box (coords authoritative); 6 hazard types | `0007` |
| B1–B3 | House scale Notable/Serious/Critical, per-feed mapping, max across feeds | `0005` |
| C1–C5 | Change semantics: event-level alertlevel, |Δmag|≥0.5, fast-moving = TC/FL/VO, 7-day corrections | `0008` |
| D1–D3 | 08:30 SGT = 00:30 UTC; first run = 24h; stale banner > 2× interval | `0002`, `0008` |
| E1–E4 | Poll 15/15/60 min; RSS-first adapter; disasters→reports later; graceful backoff | `0004` |
| F1–F3 | Fuzzy join (hazard + ±24h + ~250km, GLIDE upgrade); Situation aggregate | `0006` |
| G1–G3 | SQLite store + raw archive from slice 1; gitignored; per-feed keys | `0003` |
| H1–H4 | Dashboard layout; caps 8 events / 5 updates; quiet-day note; corrections section | _(PRD)_ |
| I1–I4 | Python 3.12; pytest on fixtures; GitHub Actions cron; UTC-at-ingest | `0001`, `0002` |

Non-ADR tuning details (caps, intervals, thresholds, dashboard layout) carry
forward into the PRD (Step B).

### A2 inconsistency fixes (2026-07-08)

1. **ReliefWeb "sitrep" vs source** — MVP `disasters` RSS carries declarations, not
   sitreps. News trigger redefined as "first disaster declaration / new GLIDE";
   casualty detail blocked on the appname API. (`ADR-0008`, `ADR-0004`, CONTEXT)
2. **PAGER-lag recall hole** — added one floor exception: admit an in-region USGS
   quake with `alert = null` when M ≥ 6.5 and depth ≤ 70 km, reconciled once PAGER
   populates. (`ADR-0008`)
3. **Stale-banner twitchiness** — per-feed grace floor of 90 min (D3 above).

### E reconciliation (2026-07-08)

Reconciled `CONTEXT.md` / `docs/PRD.md` / `prd.html` / ADRs against the shaping +
breadboard docs. Drift fixed:

1. **Two-workflow scheduling** — shaping/breadboard introduced a frequent poll
   workflow + a 00:30 report workflow, but `ADR-0002` and the PRD described only one.
   Updated `ADR-0002`, PRD scheduler bullet, and `prd.html` V3.
2. **`data`-branch persistence** — added to the PRD data model intro (`ADR-0009`);
   was only in shaping before.
3. **`Lifecycle status`** — new term (active/downgraded/deleted) added to
   `CONTEXT.md`; it appeared in the breadboard/data model but was undefined.

New open operational items (recommendations pending your confirmation):

- [ ] **E-Q1. GitHub Actions cron reliability vs the 15-min poll cadence.** Actions
  cron is best-effort and can delay/skip under load; sub-15-min is unreliable.
  **Rec:** keep GDACS/USGS at ~15-min *best-effort* (the "since last good report"
  window absorbs skips) and ReliefWeb hourly; do not depend on exact cadence.
- [ ] **E-Q2. Git churn from committing the store to `data` every poll.** A commit
  per ~15-min poll balloons history.
  **Rec:** the poll workflow force-updates `data` with a single rolling commit
  (squash), so `data` stays small; `dashboard.html` commits to `main` normally.
