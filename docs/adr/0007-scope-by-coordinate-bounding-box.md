# Geographic scope by coordinate bounding box, coordinates authoritative

Scope is Asia-Pacific natural hazards, but GDACS `country`/`iso3` can be empty
(offshore quakes) or wrong (multi-country cyclones), so a country-name list alone
mis-scopes events. We define region membership primarily by a **latitude/longitude
bounding box** (≈ lon 60°E–180°, lat 45°N–50°S); an event's coordinates are
**authoritative** over its country field, and an AP `iso3` list is a secondary
include. In-scope hazard types are EQ, TC, FL, VO, DR, WF.

## Consequences

- Offshore and multi-country events are scoped correctly by geometry.
- The box is policy and will need tuning at the edges (e.g. Indian Ocean cyclones).
- All-hazard categories (conflict, epidemics, industrial) remain out of scope — the
  three feeds cannot deliver them.
