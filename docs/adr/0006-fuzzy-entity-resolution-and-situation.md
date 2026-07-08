# Fuzzy cross-feed entity resolution and the Situation aggregate

USGS and GDACS IDs never match, GLIDE arrives days late, and the same crisis maps
many-to-one across feeds (two earthquakes → two GDACS events → one ReliefWeb
disaster). So we resolve entities **fuzzily**: same hazard type, origin time within
**±24h**, epicentre/centroid within **~250 km**, promoted to a firm match when a
shared **GLIDE** appears. Matched records are grouped into a **Situation**
aggregate — one or more Events plus any matched ReliefWeb disaster — which is the
unit the report ranks and reasons about.

## Consequences

- The report headlines Situations, not raw feed records, so a two-quake crisis is
  one story, not three.
- Fuzzy thresholds are policy and will need tuning; they live in one place.
- Within-feed duplication (ReliefWeb reposts) is deduped before cross-feed
  matching.
