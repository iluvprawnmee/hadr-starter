# Language and runtime: Python 3.12

`CLAUDE.md` sets no language, so we choose one. We build the HADR Monitor in
**Python 3.12** because the work is feed ingestion, date/timezone normalization,
and a persistent store — all well served by the standard library (`sqlite3`,
`zoneinfo`) plus mature parsing libraries — and because it fits the repo's
"deterministic checks live in `scripts/`" convention. Tests run with `pytest`
against recorded fixtures.

## Considered Options

- **Python 3.12** (chosen) — batteries-included for dates, SQLite, HTTP, feeds.
- **TypeScript/Node** — viable and strong for the HTML report, but adds a build
  step and weaker stdlib date/timezone handling for the three feed formats.
