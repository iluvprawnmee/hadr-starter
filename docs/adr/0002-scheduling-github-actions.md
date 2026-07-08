# Scheduling via GitHub Actions cron

The agent must run unattended, poll continuously, and publish at 08:30 Singapore
time. We use **two GitHub Actions cron workflows**: a **frequent poll workflow** that
ingests feeds into the store (cadence per `ADR-0004`), and a **report workflow at
00:30 UTC** (= 08:30 SGT) that builds, renders, and commits `dashboard.html`. Both
read/write the persistent store on the `data` branch (`ADR-0009`). This reuses the
`.github/` workflow surface the template already ships and needs no separate host.

## Consequences

- The public schedule is defined in UTC; the SGT anchor is a documentation fact,
  not a runtime setting — avoids the timezone-shift trap.
- GitHub Actions cron can fire late or skip under load; the "since last successful
  report" window (ADR-0008) absorbs a missed run without dropping events.
- Splitting poll from report keeps run-state churn (store commits on `data`) off the
  main-branch history, which carries only `dashboard.html`.
