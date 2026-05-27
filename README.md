# BlueDot Work Test: Course Operations Lead

Submission by Prapthi Agarwala.

## Live tool

BlueDot Round Operations (open in any browser, no install):

**Empty state:** https://yungprapz.github.io/BlueDot-WorkTest/bluedot-launch-checklist.html

**Populated demo** (Round 7 mid-round mirroring the Jamie/Cohort 11 scenario + Round 8 launching): https://yungprapz.github.io/BlueDot-WorkTest/bluedot-launch-checklist.html?demo=1

## What is here

| File | What it is |
|---|---|
| [`bluedot-launch-checklist.html`](./bluedot-launch-checklist.html) | The Part 3 systemic fix. Single-file HTML tool with three modes selectable from the top: **Dashboard** (Monday morning landing, cross-round flags, active rounds, upcoming this week, cohort heatmap), **Launching a round** (the pre-launch checklist), **Mid-round check** (weekly health check with pulse, attendance, facilitator quality, Slack signal, trends and escalation guide). State persists in localStorage. Markdown export per mode. |
| [`PART_3_WRITEUP.md`](./PART_3_WRITEUP.md) | Short writeup for the work test Google Doc. What was built, why this fix, two next-priority fixes. |
| [`docs/superpowers/specs/2026-05-26-comms-watchdog-design.md`](./docs/superpowers/specs/2026-05-26-comms-watchdog-design.md) | v2 spec. A small Vercel-hosted service that watches scheduled comms and alerts ops when an expected send did not actually go out. Independent of the email automation tool, so the Cohort 12 failure mode cannot silently repeat. |
| [`docs/superpowers/specs/2026-05-26-reminders-and-notifications-design.md`](./docs/superpowers/specs/2026-05-26-reminders-and-notifications-design.md) | v3 spec. The proactive reminder and notification layer that bridges v1 and v2. Deadline-driven reminders, watchdog failure alerts, browser notifications via Service Worker, email and SMS fallback. |
| [`index.html`](./index.html) | GitHub Pages landing page. |

## Download

- The tool as a single file: [`bluedot-launch-checklist.html`](./bluedot-launch-checklist.html) (right-click, save as)
- Everything as a zip: [Download zip](https://github.com/yungprapz/BlueDot-WorkTest/archive/refs/heads/main.zip)
- Clone:

```bash
git clone https://github.com/yungprapz/BlueDot-WorkTest.git
```

## Running the tool locally

No build step. Open the HTML file directly.

```bash
open bluedot-launch-checklist.html
```

State persists in your browser's localStorage. Export to markdown at any time from the header.

## Privacy

The tool runs entirely client-side. No backend, no accounts, no tracking. Everything you enter (round details, participant counts, notes, issues) stays in your browser.
