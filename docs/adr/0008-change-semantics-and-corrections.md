# Change semantics: the diff engine and corrections policy

The product's hard requirement — "stay quiet when nothing has changed" — forces an
explicit definition of News. Against the persistent store (ADR-0003), an in-scope
Situation is re-surfaced only on: a **new Event**, an **escalation** (alert level
up), a **material revision** (USGS |Δmag| ≥ 0.5, or any GDACS alert-level colour
change), a **correction** (an event we headlined is downgraded or deleted), the
**first ReliefWeb signal** on a known disaster, or a **new Episode for a
fast-moving hazard** (cyclone/flood/volcano). Static re-issues, Nth signals with no
new substance, and sub-floor churn are not News.

## PAGER-lag fallback (earthquakes)

USGS PAGER `alert` is `null` for the first 20–30+ min after a quake, so a
just-happened event can sit below the PAGER-yellow floor at report time. To close
that recall hole, an in-region USGS quake is admitted **even with `alert = null`**
when **magnitude ≥ 6.5 and depth ≤ 70 km**. Such an event is reported provisionally
(house level from magnitude/depth) and reconciled once PAGER populates — a
downgrade to below-floor then becomes a correction.

## ReliefWeb news trigger in the MVP

Until the appname-gated `reports` API is available (ADR-0004), the MVP reads the
`disasters` RSS feed, which carries **declarations, not sitreps**. So "first
ReliefWeb signal" concretely means **first disaster declaration / new GLIDE**;
casualty figures and sitrep counts are absent until the API path is live.

## Deletion detection is gated on successful polls, not elapsed time

A feed dropping an event (USGS records vanish; GDACS `istemporary` events
disappear) has no tombstone, so deletion is inferred from absence. But absence is
ambiguous when *we* failed to poll — and polling runs on best-effort GitHub Actions
cron (`ADR-0002`), which delays and skips. So deletion is **not** timed off the
wall clock. An Event transitions to `deleted` only after **N consecutive
*successful* polls of its source feed** (per `feed_health`) each returned the feed
and omitted the event, while it is still inside the report window. Missed or failed
polls do not count toward N and never trigger a deletion. This keeps a slow-CI
morning from emitting false retractions — the corrections feature stays trustworthy
precisely when the infrastructure is flaky.

## Consequences

- GDACS event-level `alertlevel` drives the severity floor and headline; episode
  level drives fast-moving-hazard episode news.
- The severity floor (GDACS Orange+ / PAGER yellow+ / ReliefWeb declaration) has
  exactly one exception: the M≥6.5, depth≤70 km PAGER-lag fallback above.
- Corrections are issued for 7 days after an event was last headlined, even when
  the change is downward — the report retracts, it does not go silent.
- The report window is "since the last successful report" in UTC (ADR-0002), so a
  missed run never drops events.
