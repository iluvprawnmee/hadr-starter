---
shaping: true
---

# R2.1 Spike: Cross-feed entity resolution

## Context

R2.1 (fuzzy cross-feed resolution → Situations) is flagged in every shape (A5/B5/C5)
— we can name "match on hazard + time + geo, upgrade on GLIDE" but have not spelled
out the concrete mechanism. This is the one shared unknown blocking shape selection.
This spike resolves it from the feed field structures in `feeds/*.md` and
`docs/blindspots.md`.

## Goal

Describe concretely: which fields identify and join records across GDACS, USGS, and
ReliefWeb; the per-hazard tolerances that separate "same crisis" from "different
crises"; and how the join runs incrementally in the Shape C entity tracker — such
that R2.1 can move from ⚠️ to a concrete mechanism.

## Questions

| # | Question | Finding |
|---|----------|---------|
| **ER-Q1** | What stable identifiers exist per feed, and which survive polls/revisions? | GDACS: `(eventtype, eventid)` stable; `episodeid` changes per update; `glide` usually empty at first. USGS: `id` (network-prefixed, e.g. `ci41287863`) stable; `ids` is a comma-list of all network ids for the same quake (e.g. `,ci41287863,us6000tafd,`). ReliefWeb: `glide` (e.g. `EQ-2026-000093-VEN`) + the `link` URL, both stable. **No shared key exists across feeds** — GDACS `eventid` ≠ USGS `id` ≠ ReliefWeb GLIDE. |
| **ER-Q2** | How to match a NEIC-sourced GDACS earthquake to the same USGS quake? | GDACS EQ carries `source: "NEIC"` (the USGS network) and the same origin time + coordinates USGS reports. There is no id overlap, but the physical event is identical, so match on **origin time + epicentre proximity**. (USGS `ids` sometimes contains a `us…` id that GDACS's detail endpoint also references, but relying on that is fragile — geo+time is the robust join.) |
| **ER-Q3** | What time/distance tolerances separate "same" from "different"? | Tolerances are **hazard-specific** — a single ±24h/250km rule is too loose for quakes and too tight for cyclones. See the table below. Earthquakes have precise origin times/locations (tight); cyclones move hundreds of km over days (loose, prefer name); floods/drought are areal and slow. |
| **ER-Q4** | How does ReliefWeb tie in, given it lags days and describes areal crises? | ReliefWeb rarely shares coordinates; it carries country + GLIDE + hazard type in the description. Join a ReliefWeb disaster to detection Events by **GLIDE when present** (deterministic), else by **hazard type + affected country/iso3 + a wide time window** (±72h). This is intentionally loose because ReliefWeb is the "one story" umbrella. |
| **ER-Q5** | How to represent one-to-many (two quakes → one ReliefWeb disaster)? | The Situation aggregate. A ReliefWeb disaster attaches to *every* detection Event matching its hazard+country+window, so the Venezuela M7.1 and M7.5 both link to the single `EQ-2026-000093-VEN` Situation. Situation `house_severity` = max across members. |
| **ER-Q6** | How does the join run incrementally as records arrive across polls (Shape C)? | On each incoming normalized record: (1) compute a **blocking key** = `hazard_type + coarse geo-cell + time-bucket` to limit comparisons; (2) try a **deterministic GLIDE join** against active Situations; (3) else **score** candidate Events in the block by the hazard tolerances; (4) attach to the best match above threshold, or create a new Event/Situation. A later GLIDE promotes/merges fuzzy matches. |

## Per-hazard matching tolerances

**Provisional — starting points from reasoning, to be calibrated against fixtures
before V2 (see "Calibration" below).** These are the Stage-2 (crisis-aggregation)
windows; Stage-1 physical-event dedup uses tighter values (see "Two matching
stages").

| Hazard | Time window | Distance | Extra | Rationale |
|--------|-------------|----------|-------|-----------|
| Earthquake (EQ) | ±30 min | ≤ 150 km | \|Δmag\| ≤ 1.0 | Precise origin/time; window absorbs revisions & late PAGER, not distinct quakes. Dedupes GDACS EQ vs USGS. |
| Cyclone (TC) | ±24 h | ≤ 500 km centroid | prefer name/basin match | Tracks move far between episodes; name is the strongest signal. |
| Flood (FL) | ±48 h | ≤ 250 km | country overlap | Areal, slower onset. |
| Volcano (VO) | ±24 h | ≤ 50 km | — | Fixed location; tight distance. |
| Drought (DR) | ±14 d | country match | — | Slow-onset, areal; time/geo are coarse. |
| Wildfire (WF) | ±48 h | ≤ 250 km | country overlap | Areal, fast-moving. |
| ReliefWeb → any | ±72 h | country/iso3 match | GLIDE overrides all | Umbrella record; deliberately loose. |

## Recommended mechanism (resolves R2.1)

1. **Normalize** each feed record to a match candidate: `hazard_type, occurred_at
   (UTC), lat, lon, [magnitude, depth], country, iso3, glide, feed_keys`.
2. **Block** by `hazard_type + geo-cell (~1°) + time-bucket` to keep comparisons cheap.
3. **Deterministic GLIDE join** first: records sharing a GLIDE are the same Situation.
4. **Two matching stages, not one** (the key correction):
   - **Stage 1 — physical-event dedup (tight):** is the incoming record the *same
     physical event* as an existing one — e.g. the same quake from GDACS(NEIC) and
     USGS? EQ tolerances ~±90 s, ≤50 km, compatible `magType`. A match merges into
     one Event; it must **not** merge two distinct quakes.
   - **Stage 2 — crisis aggregation (loose):** group distinct-but-related Events
     into one Situation using the per-hazard windows in the table above (EQ ±30 min
     / 150 km, etc.). Deliberately loose — may span an aftershock sequence.
   Conflating these was the original error: one threshold set cannot be both tight
   enough to keep separate quakes apart and loose enough to group a crisis.
5. **Situation aggregation**: Stage-2 groups Events into a Situation; a ReliefWeb
   disaster attaches to all Events sharing hazard + country + window (one-to-many).
6. **GLIDE upgrade**: when a GLIDE later appears, confirm/merge the fuzzy grouping and
   record it as the firm identity.
7. **Within-ReliefWeb dedup** (R2.2) runs before step 5: collapse reposts of one
   disaster by GLIDE/link.

## Calibration (before V2)

The tolerances above are reasoned starting points, not measured. Before the V2
resolver is locked, **calibrate them against the recorded fixtures (`R8.1`)** —
especially aftershock sequences and the Venezuela M7.1/M7.5 case — and track two
error rates:

- **false-merge** — two distinct crises collapsed into one Event/Situation (risks
  corrupting `house_severity(max)` and the ranked headline);
- **false-split** — one crisis fragmented into several Situations.

Tune Stage-1 (dedup) and Stage-2 (aggregation) independently. This calibration is a
V2 build task, tracked as `E-Q`-style follow-up, not a blocker on the shape.

## Acceptance

Complete — we can now describe, per hazard, which fields join records across the
three feeds, the **two-stage** matching (tight physical-event dedup, then loose
crisis aggregation), how one-to-many crises aggregate into a Situation, and how the
join runs incrementally in the entity tracker. R2.1's mechanism is concrete; its
tolerances are provisional pending fixture calibration in V2.
