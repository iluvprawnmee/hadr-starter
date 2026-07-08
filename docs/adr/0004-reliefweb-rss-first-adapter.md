# ReliefWeb: RSS first, behind an adapter, upgrade to the reports API later

Since 1 Nov 2025 the ReliefWeb API requires a pre-approved `appname` that may not
be granted within the build week, and the substance we want (casualty figures,
sitreps, appeals) lives in the `reports` content type, reachable only via the API.
We therefore **build against the no-approval `disasters` RSS feed now**, behind a
**source adapter interface**, so the `reports` API can slot in later without
touching the pipeline.

## Consequences

- MVP gets official naming and GLIDE from the RSS `disasters` feed, but lacks
  casualty/sitrep detail — a **known, documented limitation** until the appname
  clears.
- The API path must budget calls against the 1,000/day cap (1,000 entries/call);
  the RSS path has no such budget.
- Recorded as a deliberate deviation so a future reader does not "fix" the RSS
  choice without knowing the appname constraint.
