# Fuzzy cross-feed entity resolution and the Situation aggregate

USGS and GDACS IDs never match, GLIDE arrives days late, and the same crisis maps
many-to-one across feeds (two earthquakes → two GDACS events → one ReliefWeb
disaster). So we resolve entities **fuzzily**, in **two distinct stages** (one
threshold set cannot serve both):

1. **Physical-event dedup (tight)** — decide whether two records are the *same
   physical event* (e.g. a quake from GDACS(NEIC) and USGS). Tight tolerances.
2. **Crisis aggregation (loose)** — group distinct-but-related Events into one
   **Situation**, promoted to a firm grouping when a shared **GLIDE** appears.

Per-hazard tolerances live in `SPIKE-entity-resolution.md` and are **provisional
until calibrated against the recorded fixtures before V2**. The Situation — one or
more Events plus any matched ReliefWeb disaster — is the unit the report ranks and
reasons about.

## Consequences

- The report headlines Situations, not raw feed records, so a two-quake crisis is
  one story, not three.
- Fuzzy thresholds are policy and will need tuning; they live in one place.
- Within-feed duplication (ReliefWeb reposts) is deduped before cross-feed
  matching.
