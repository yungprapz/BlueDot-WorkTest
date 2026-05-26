# BlueDot Work Test: Course Operations Lead

Submission by Prapthi Agarwala.

## Live tool

Pre-round launch checklist (open in any browser, no install):

**https://yungprapz.github.io/BlueDot-WorkTest/**

## What is here

| File | What it is |
|---|---|
| [`bluedot-launch-checklist.html`](./bluedot-launch-checklist.html) | The Part 3 systemic fix. Single-file HTML tool, opens via `file://` or hosted. Seven sections per cohort, round summary, edge case prompts, issue log, failure response reference, markdown export, localStorage persistence, dark mode. |
| [`PART_3_WRITEUP.md`](./PART_3_WRITEUP.md) | Short writeup for the work test Google Doc. What was built, why this fix, two next-priority fixes. |
| [`docs/superpowers/specs/2026-05-26-comms-watchdog-design.md`](./docs/superpowers/specs/2026-05-26-comms-watchdog-design.md) | Next-iteration spec. A small Vercel-hosted service that watches scheduled comms and alerts ops when an expected send did not actually go out. Independent of the email automation tool, so the Cohort 12 failure mode cannot silently repeat. |
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
