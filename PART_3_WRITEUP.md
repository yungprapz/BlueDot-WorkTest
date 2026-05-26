# Part 3: One systemic fix

## What I built

A pre-round launch checklist tool. Single HTML file, runs in any browser, no install, state persists in localStorage so a teammate can pick it up where someone else left off.

Each round of cohorts gets one checklist instance. For every cohort the checklist covers seven sections that have to be true before session 1: welcome email, facilitator readiness, calendar invites, Slack channel, roster reconciliation, course materials, tech check. Items are tickable, each can carry a note, and any item can spawn an entry in a round-level issue log so a near-miss does not vanish.

A round summary view shows per-cohort progress and status (Ready, In progress, Not ready, Not started) at a glance. An edge case panel surfaces the seven situations most likely to be missed during launch (new facilitators, deferrals, cohort switches, public holidays, timezones). A failure response reference table is one click away from any view, so if something does break the next action is already written down. The whole checklist exports to a single markdown file for archiving or attaching to a post-round retro.

Artifact: `bluedot-launch-checklist.html` (link in the submission)

## Why this fix

The Cohort 12 welcome email automation failed silently. Five participants told us. The true number affected was probably closer to ten. The root cause was not "the automation tool is bad", it was "the launch ritual relied on the automation working and had no human checkpoint". The same pattern would have caught a wrong Zoom link, a stale facilitator brief, or a missing Slack channel.

I built a tool rather than a checklist document because operations work that lives in a doc tends to get forgotten or copied stale. A live tool with state persistence makes the right behaviour the easy behaviour, and the export gives us a permanent record per round without anyone maintaining one by hand.

Scope choices made to ship inside the budget: no backend, no auth, no integrations. localStorage instead of a shared database. Vanilla JS in one file. These are reversible later if BlueDot wants this hosted with sign-in and audit history. They are the right defaults for now because they cut every blocker between "I built it" and "a teammate uses it Monday morning".

## Two next-priority fixes

1. **Pulse survey alerting on trending-negative cohorts.** The Cohort 11 situation with Jamie was visible in pulse surveys for weeks 2 and 3 before it surfaced as a complaint with a public-posting threat. A weekly digest that flags cohorts whose pulse scores drop more than one standard deviation, or where two of three surveys land amber or red, would let us intervene a week earlier. Build cost is small if the survey data is already exportable.

2. **A weekend-inbound triage rhythm.** Five of the nine items in this pile landed between Friday evening and Sunday night. By Monday morning two were already past the point where the cheapest response was possible (Sarah Chen, Hannah Liu both wanted answers by Tuesday EOD). Either a Saturday-morning 30-minute triage check by a designated on-call, or a shared inbox with auto-acknowledgement plus a Sunday-evening prep block on the Course Ops Lead calendar, would reduce Monday-morning surface area meaningfully without adding headcount.
