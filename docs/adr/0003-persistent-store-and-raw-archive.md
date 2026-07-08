# Persistent event store and raw-payload archive from the first slice

The feeds are rolling windows, not logs: events age out, GDACS `istemporary`
events vanish, and USGS deletions leave no tombstone. An agent that polls and
discards can never detect change, deletion, or test against yesterday. Therefore,
**from the very first slice**, we keep our own persistent normalized event store
(**SQLite**) and archive **every raw payload** as a timestamped file before
parsing.

## Consequences

- Change detection, corrections, and entity resolution all read from our store,
  not from a single live snapshot.
- The raw archive doubles as the source of recorded fixtures for deterministic
  replay and as protection against undocumented schema drift (raw is kept even
  when parsing breaks).
- The store and a rolling raw archive persist across ephemeral CI runs by being
  committed to a dedicated `data` branch (see `ADR-0009`); curated fixtures and
  `dashboard.html` live on the main branch.
