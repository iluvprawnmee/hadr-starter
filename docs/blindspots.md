# Blindspot pass: GDACS + USGS + ReliefWeb

Unknown unknowns about the three feeds and the HADR monitoring problem, found on
7 Jul 2026 before writing the PRD. Complements the known unknowns already listed
in `feeds/*.md` — nothing here repeats those. ReliefWeb limits verified against
https://apidoc.reliefweb.int/ the same day.

## The biggest one: these are rolling windows, not logs

Both the GDACS event list and the USGS `all_day` feed show *current state*, not
history. Events age out of the window, GDACS `istemporary` events can vanish
entirely, and USGS deletions just disappear from the feed — there is never a
tombstone. If the agent polls and discards, it can never reconstruct what it
saw, never detect a deletion, and never test against yesterday.

**Consequence:** the agent needs its own persistent event store and a
raw-payload archive from the very first slice, not as later hardening. The
archive also solves the Wednesday demo problem: disasters are rare, so the live
feeds may be boring during the demo — recorded fixtures allow a deterministic
replay of a real day (this belongs in `scripts/`).

## The product is a diff engine, not a fetcher

The hardest requirement in the README is hidden in "stays quiet when nothing
has changed." That forces a *definition* of change: is a new episode of a known
cyclone news? A magnitude revision from 7.3 to 7.1? A tenth ReliefWeb sitrep on
a week-old disaster? An alert downgrade? Each answer is an editorial policy,
and the state machine implementing it is the actual system — fetching is the
trivial part.

Related: what does the report say when yesterday's headline event is revised
away (USGS deletes it, GDACS downgrades Red → Orange)? A credible sitrep issues
corrections; that policy must be decided up front.

## Timestamps will silently corrupt the 08:30 report

Three feeds, three time formats:

| Feed | Format | Trap |
| --- | --- | --- |
| GDACS | Naive ISO string, no timezone suffix (`"2026-07-06T11:29:36"`) | It is UTC, but any parser will happily read it as local time — shifting everything by 8 hours in Singapore |
| USGS | Epoch milliseconds | Easy to misread as seconds |
| ReliefWeb RSS | RFC 822 (`Wed, 24 Jun 2026 00:00:00 +0000`) | Different parser again |

An 08:30 SGT report window built on mis-parsed GDACS times is wrong in a way
nothing crashes on. Normalize everything to UTC at ingest, and define the
report window explicitly (which 24 hours, data as-of when).

## GDACS

- **Alert levels measure modeled humanitarian impact, not physical size** —
  population exposure × vulnerability. That is why they are the useful signal,
  but also why they are revised as the model reruns, and why Green is ~95% of
  feed volume. The severity floor decision is really a recall/precision
  decision.
- **The event/episode model varies wildly by hazard.** A cyclone is one event
  with dozens of episodes over a week as the forecast track updates; a drought
  runs for months and episodes can reactivate. "New event" logic tuned on
  earthquakes will spam on cyclones.
- **Geometry is a point even for areal hazards.** A flood is not a point;
  actual polygons live behind a separate geometry endpoint. The `country` /
  `iso3` fields can be empty (offshore quakes) or wrong for multi-country
  cyclones — "where" needs its own logic.
- **Multi-hazard ≠ all-hazard.** No landslides, epidemics, heatwaves,
  industrial accidents, or conflict. If HADR in the PRD means more than
  natural hazards, these three feeds cannot deliver it — scope that
  explicitly.
- **The schema is semi-documented and shifts without notice.** The Swagger
  page at `gdacs.org/gdacsapi/swagger` is nearly empty, which is
  representative. Store the raw JSON alongside the parsed model so schema
  drift breaks parsing, not history.

## USGS

- **The feed's composition is an instrument-density artifact.** `all_day` is
  dominated by M1–2 California/Alaska quakes because that is where the sensors
  are; global completeness is only ~M4.5. "208 earthquakes today" is
  meaningless as a world statement, and *absence* of small events in, say,
  Central Asia means nothing. Choosing `4.5_day` vs `all_day` is an editorial
  decision, not a technical one.
- **Magnitude is a bad humanitarian proxy.** M7.8 offshore can matter less
  than M6.0 shallow under a city. The humanitarian signal is the `alert` field
  (PAGER: green/yellow/orange/red, estimating fatalities and economic loss) —
  but PAGER arrives 20–30+ minutes after origin and gets revised, so an early
  poll sees `null` where a red will later appear. Depth and `magType` matter
  too (mb/ML/Mw are not directly comparable).
- **`tsunami: 1` does not mean a tsunami happened** — it means the event falls
  in a NOAA-warned zone. Real tsunami warnings are a separate source
  (tsunami.gov / PTWC) not currently in scope.
- **USGS is earthquakes only.** For every other hazard — including in the US —
  GDACS is the sole detection feed. That single point of failure is worth
  naming in the PRD.

## ReliefWeb

- **`disasters` is the wrong content type for substance.** The disasters RSS
  is a slow, curated registry — good for GLIDE numbers and official naming,
  useless for detail. The actual flow of situation reports, casualty figures,
  and appeals is the `reports` content type: hundreds per day, richer, faster.
  "How bad, who is affected" lives in reports, not disasters. The API (once
  the appname clears) filters reports by disaster, country, and format; the
  RSS does not.
- **Three date fields, three meanings**: `date.created` (posted to ReliefWeb),
  `date.original` (the document's own date), `date.changed`. Polling on the
  wrong one double-counts updated documents or misses late-posted ones.
- **Selection bias by design.** ReliefWeb reflects the *international
  humanitarian system's attention*, not the world's disasters. A Japan M7
  rarely appears (high domestic capacity, no international response); conflict
  and slow-onset crises dominate the long tail. "GDACS Red, ReliefWeb silent"
  is a normal state meaning either "too early" or "no international response
  needed" — the report should treat those differently.
- **The 1,000 calls/day budget bites in non-obvious ways** (verified: 1,000
  calls/day, 1,000 entries/call). Polling is cheap, but a naive "one API call
  per known disaster per run" loop, or backfill experimentation, burns it
  fast. Budget calls per scheduled run explicitly.
- **In-feed duplication**: the same sitrep is often posted by multiple
  sources; dedup *within* ReliefWeb comes before cross-feed matching.

## Cross-feed traps beyond `feeds/*.md`

- **Severity normalization is a false friend.** GDACS colors and PAGER colors
  look identical but measure different things (multi-hazard impact model vs
  earthquake loss estimate); ReliefWeb has no severity at all. A unified
  "severity" column will quietly lie. Pick a house scale and document the
  mapping as policy.
- **Entity resolution is one-to-many, not one-to-one.** The Venezuela example
  in `feeds/reliefweb.md` shows it: *two* USGS earthquakes (M7.1, M7.5) →
  probably two GDACS events → *one* ReliefWeb disaster. GLIDE arrives days
  late and USGS/GDACS IDs never match, so the practical join is fuzzy: hazard
  type + time window + geographic distance, upgraded to GLIDE when it appears.
- **The big-day problem is the quiet-day problem's evil twin.** A real Red
  event produces GDACS episode churn plus a flood of ReliefWeb reports. The
  summarization step needs caps and prioritization, or the one morning that
  matters is the morning the pipeline chokes.

## Decisions to pin in the PRD and CLAUDE.md

The feeds cannot answer these; they are policy:

1. **Scope** — hazard types, geography (global, or Asia-Pacific given the SGT
   anchor?), severity floor (e.g., GDACS Orange+ or PAGER yellow+, plus any
   ReliefWeb-declared disaster).
2. **Change semantics** — what counts as news per feed: new event, new
   episode, escalation, revision, new report on an old disaster.
3. **Corrections policy** — how the report handles downgraded or deleted
   events it already headlined.
4. **Report window** — defined in UTC, with explicit data-as-of semantics, and
   a stale-data banner (last successful fetch per feed) on a feed-down
   morning.
5. **Operational budget** — per-feed polling cadence, backoff, and a ReliefWeb
   call budget per scheduled run.
6. **Data policy** — archive every raw payload; normalize timestamps to UTC at
   ingest; test against recorded fixtures, not live feeds.
7. **Feed quirks worth pasting into CLAUDE.md** so no session rediscovers them:
   naive GDACS timestamps, USGS `ids` handling, ReliefWeb `date.created` vs
   `date.original`, PAGER lag, rolling-window deletions.
