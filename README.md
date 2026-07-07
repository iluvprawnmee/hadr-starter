# HADR Monitor

A monitoring agent for humanitarian assistance and disaster response (HADR).

## The end state

By Wednesday afternoon this repository contains an agent that:

- watches live disaster feeds — GDACS, USGS and ReliefWeb (see `feeds/`)
- filters out the noise and assesses what remains: what happened, where, how bad, who is affected
- publishes a morning situation report to `dashboard.html` at 08:30 Singapore time
- runs on a schedule, unattended, and stays quiet when nothing has changed

How it does any of that is not specified anywhere in this repository. That is the course.

## The three days

1. **Plan** — interrogate the feeds, write the PRD, cut it into vertical slices
2. **Autonomy** — build the first slice, write a skill, wire up the 08:30 routine, launch the overnight loop
3. **Trust** — review code you didn't write, harden the pipeline, demo

## Artefacts expected by the end

`prd.html` · `system-view.html` · `implementation-notes.md` · `dashboard.html` · `goal.md` · at least one skill

## Repository layout

What ships in this template, and what each piece is for:

| Path | What it is |
| --- | --- |
| `CLAUDE.md` | Project conventions the agent must follow — language, test command, coding conventions, deviations policy. Fill this in before your first prompt; an empty file is a decision too. |
| `implementation-notes.md` | The agent's working log — decisions, open questions and deviations, one entry per working block. You review it. |
| `feeds/` | Field notes on the three live sources: `gdacs.md`, `usgs.md`, `reliefweb.md`. Each has a verified endpoint, a truncated example response, and open questions to resolve during planning. |
| `scripts/` | Deterministic checks. Anything that must give the same answer twice lives here rather than in a prompt. |
| `skills/` | Skills you write on Day 2, one folder per skill: a `SKILL.md`, supporting assets, and a note on which model each step should use. |
| `docs/solutions/` | One learning per file (`YYYY-MM-DD-short-slug.md`). When a problem costs you more than ten minutes, the fix goes here so no future session pays for it twice. |
| `.github/` | Issue templates (`slice`, `skill`) and the @claude review + sitrep workflows. |
| `.gitignore` | Keeps generated reports and secrets out of the repo; `dashboard.html` is the committed exception. |

The feeds, the conventions, and the notes are given. The agent that turns them into an 08:30 situation report is what you build.

## Day 1 setup

1. Sign in to Claude Code with your Team seat
2. Create your own repository from this template, then clone it
3. Run `/install-github-app` so @claude reviews your pull requests from Day 2
4. Install OpenCode and sign in with your Go key

Fill in `CLAUDE.md` before your first prompt.
