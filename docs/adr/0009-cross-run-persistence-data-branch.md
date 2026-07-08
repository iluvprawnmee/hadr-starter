# Cross-run persistence: commit the store to a `data` branch

The agent runs on ephemeral GitHub Actions runners, but the diff engine depends on
a persistent event store surviving between runs (`ADR-0003`). We persist it by
**committing the SQLite store and a rolling raw-payload archive to a dedicated
`data` branch** on each run. The repo itself is the persistence layer — no external
infrastructure or secrets.

This **supersedes** the earlier "store and archive are gitignored" consequence of
`ADR-0003`.

## Considered Options

- **`data` branch commit** (chosen) — self-contained, auditable history, zero infra.
  Cost: git noise, and the archive must be pruned to a rolling window to bound repo
  size.
- **Actions cache / artifacts** — keeps the repo clean, but cache eviction (7-day
  retention, size limits) can silently drop history, which is fatal for a diff
  engine that needs continuity.
- **External storage** (object store / libSQL) — robust, but adds a dependency and
  secrets — disproportionate for a build-week project.

## Consequences

- `dashboard.html`, code, fixtures, and docs live on `main`; the store + rolling
  archive live on `data`, keeping run-state churn off the main history.
- The archive is pruned to a rolling window on `data`; long-term raw retention, if
  needed, is a separate concern.
