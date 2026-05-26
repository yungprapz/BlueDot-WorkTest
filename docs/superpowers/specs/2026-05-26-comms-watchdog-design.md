# Comms Watchdog: automated verification that scheduled comms actually went out

Status: draft, awaiting review
Author: Prapthi Agarwala (with Claude)
Date: 2026-05-26
Companion to: bluedot-launch-checklist.html (v1, shipped today)

## What this is and is not

The v1 launch checklist is a human backstop. It catches missed comms only if a human runs the checklist and only at one moment in time (Monday morning of launch week).

This spec defines a **second, independent system**: a watchdog that knows what comms are supposed to go out, when, and to whom, and that verifies each one actually went, on schedule. When a comm is late or missing, the watchdog pages the on-call ops person before participants notice.

The watchdog is not:

- A replacement for the email automation tool. The automation tool still sends. The watchdog only verifies.
- A replacement for the v1 checklist. v1 covers everything that cannot be machine-verified (facilitator readiness, materials quality, edge cases).
- A new sender of comms. It alerts ops; it does not send participant-facing email.

The whole point is **defense in depth**. The Cohort 12 failure happened because we had one system responsible for sending and that same system was also implicitly responsible for telling us if it failed. The fix is to split those jobs.

## Goals

1. Detect any scheduled comm that did not go out, within an hour of its scheduled send time
2. Detect comms that went out but to the wrong recipients (missing participants, extra recipients)
3. Surface delivery failures (bounces) per cohort, per round
4. Route alerts to the on-call ops person via a channel they actually watch (Slack, with email backup)
5. Be cheap to host (Vercel free tier) and cheap to maintain (no DB to back up, declarative schedule)
6. Be testable end-to-end without sending real participant emails

Non-goals:

- Replacing or wrapping the email automation tool
- Building a UI for participants
- Multi-tenant accounts
- Anything related to course content, facilitator quality, or pulse surveys (these are different problems with different solutions)

## What we watch

Five comm types, in priority order. Each has a known schedule relative to session 1 of a cohort.

| Comm | When it sends | Audience | Source of truth |
|---|---|---|---|
| Welcome email | T-48h before session 1 | All cohort participants | Email automation tool delivery log |
| Session reminder | T-2h before each session (weeks 1-5) | All cohort participants | Email automation tool delivery log |
| Weekly materials drop | Day-of, week N session, T-24h | All cohort participants | Email automation tool delivery log |
| Calendar invite | T-7d before session 1 | All cohort participants | Google Calendar API events list |
| Slack channel access | T-72h before session 1 | All cohort participants | Slack API conversation members |

v2.0 ships welcome email verification only. The others phase in as v2.1, v2.2, v2.3.

## Recommended architecture

A small Vercel-hosted service. Three components, each with one job.

```
┌─────────────────────┐
│  Schedule registry  │   ← declarative config (YAML or Google Sheet)
│  what / when / who  │     "Cohort 12 welcome email at 2026-05-25 09:00 UTC"
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Verifier (cron)    │   ← Vercel Cron, runs every 15 minutes
│  ┌───────────────┐  │     At each tick:
│  │ Pull adapters │──┼──→ 1. Get scheduled comms due in last hour
│  │ Email API     │  │     2. For each: query source of truth
│  │ Calendar API  │  │     3. Compare expected vs observed
│  │ Slack API     │  │     4. Emit verification result
│  └───────────────┘  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐     ┌──────────────────┐
│  Alert router       │ ──→ │ Slack channel    │
│  (only on failure)  │ ──→ │ Ops Lead email   │
└─────────────────────┘     └──────────────────┘

           Plus:
┌─────────────────────┐
│  Status dashboard   │   ← /status page, public read or auth-gated
│  (last 14 days)     │     Expected vs verified vs failed per cohort
└─────────────────────┘

           Plus:
┌─────────────────────┐
│  Heartbeat          │   ← External cron ping (cron-job.org or similar)
│  (dead-man switch)  │     Pages ops if the watchdog itself stops running
└─────────────────────┘
```

### Why this shape

- **Cron + serverless** means zero idle cost. Free tier is sufficient at BlueDot's scale (one round per six weeks, six cohorts per round, five comm types per cohort = ~30 comms per round to verify).
- **No DB.** State is "what was scheduled" (config) plus "what did the source of truth say at check time" (transient). The only thing that needs to persist is "we already alerted on this failure" to prevent alert spam, and that's a tiny KV (Upstash via Vercel Marketplace, or a JSON blob in Vercel Blob).
- **Declarative schedule** means non-engineers can update it. Two formats supported: a YAML file in the repo (eng-edited, version controlled) and a Google Sheet (ops-edited, friendlier). Both compile to the same internal shape.
- **External heartbeat** is the second-order watchdog. If the verifier itself dies, an outside cron will page within 5 minutes.

### Pull vs push

For email automation: pull from the tool's API. Push (the tool webhooks the watchdog on every send) would be more reliable but assumes the tool supports webhooks and that those webhooks can themselves be trusted to fire. That's the original failure mode. Pull is safer because it doesn't depend on the system being watched.

For calendar and Slack: pull, same reasoning.

If a specific tool later proves to have reliable outgoing webhooks, the design accepts push as an additive signal, but never as the only signal.

## Verification rules (v2.0: welcome email only)

For each cohort with a welcome email scheduled in the last hour:

1. **Send attempted?** Email tool's delivery log contains records tagged with the cohort identifier
2. **Send complete?** All roster entries for the cohort have at least one delivery record
3. **Send successful?** All delivery records show status `delivered` (or tool's equivalent), zero `bounced` / `failed` / `dropped`
4. **Send on time?** Latest delivery timestamp is within ±30 minutes of scheduled time
5. **Send to right audience?** Set of recipients in delivery log equals set of emails in roster for that cohort

A failure of any check fires an alert. The alert names which check failed and why.

```yaml
# Example alert payload
cohort: "Cohort 12"
round: 8
comm: "welcome-email"
scheduled_at: "2026-05-25T09:00:00Z"
status: "failed"
checks:
  send_attempted: pass
  send_complete: FAIL
  send_successful: pass
  send_on_time: pass
  audience_match: FAIL
details:
  missing_recipients:
    - alice@example.com
    - bob@example.com
    - carol@example.com
    - dave@example.com
    - evan@example.com
recommended_action: "Send manually using the welcome email template to the 5 missing recipients. See failure response reference in the checklist tool."
```

## Schedule registry format

A single source of truth. Engineers prefer YAML in the repo; ops prefer a Google Sheet. The watchdog reads either.

```yaml
# config/schedule.yml
round: 8
course: "Technical AI Safety"
session_1_at: "2026-06-15T18:00:00+01:00"
cohorts:
  - id: c7
    label: "Cohort 7"
    session_local_time: "2026-06-16T12:00:00+01:00"
    facilitator: aisha
  - id: c12
    label: "Cohort 12"
    session_local_time: "2026-06-16T18:00:00+01:00"
    facilitator: ben
comms:
  - type: welcome-email
    offset_from_session_1: "-48h"
  - type: session-reminder
    offset_from_each_session: "-2h"
  - type: weekly-materials
    offset_from_each_session: "-24h"
  - type: calendar-invite
    offset_from_session_1: "-7d"
  - type: slack-access
    offset_from_session_1: "-72h"
```

The compiled schedule generates one verification job per (cohort × comm × instance). The watchdog stores these as concrete `(cohort, comm, expected_at)` triples in memory at boot.

## Alert routing

Alerts route to two destinations, both required:

- **Slack channel `#ops-watchdog`** - primary, immediate
- **Course Ops Lead email** - secondary, ensures alert survives if Slack is down or muted

Each alert includes a unique `(cohort, comm, scheduled_at)` key. If the same failure persists across multiple check cycles, the alert escalates after 1 hour (extra ping naming the time elapsed, CCs Course Lead) and again after 4 hours (pages CEO). State of "already alerted" lives in KV with a 48-hour TTL.

A Slack message includes a "Mark as handled" button. Clicking it (via a tiny Slack interaction endpoint) suppresses re-alerts for that key. Manual recovery is expected to happen in the actual email tool, not in the watchdog.

## Status dashboard

Lives at `/status` on the watchdog. Read-only.

Shows for the last 14 days:

- Expected comms count
- Verified-pass count
- Verified-fail count (with click-through to alert detail)
- Last verifier run timestamp
- Heartbeat status

The dashboard is the daily-check artifact for Course Ops Lead. If it stays green, the launch is on rails. If it goes red, the alert has already fired.

## Integration with v1 launch checklist

When a watchdog verification has run for a cohort's welcome email, the v1 checklist tool fetches the result on load and displays it inline next to the relevant item.

- If `verified: pass` - the item shows a small green tag "Verified by watchdog 2 hours ago" and the checkbox auto-ticks (with override allowed)
- If `verified: fail` - the item shows a red tag with the failure summary and links to the alert
- If `verified: unknown` - no change, manual tick as today

This makes v1 and v2 complementary, not redundant. v1 handles everything machines cannot verify; v2 handles everything they can.

The fetch is a single GET to the watchdog's `/api/verification?round=8&cohort=c12` endpoint. If the watchdog is unreachable, v1 silently degrades to its v1 behavior. No hard dependency.

## Hosting and stack

| Choice | Why |
|---|---|
| Vercel for hosting | Cron jobs, serverless functions, free tier sufficient |
| Next.js App Router | Routing for dashboard, API routes for verification endpoint, single deploy unit |
| TypeScript | Verification rule contracts benefit from explicit types |
| Upstash Redis (via Vercel Marketplace) | KV for alert deduplication; tiny footprint |
| Vercel Blob | Optional storage for schedule history and alert audit log |
| Vercel Cron | Verifier scheduler. One job at `*/15 * * * *`. |
| External cron (cron-job.org or similar) | Heartbeat pinging the watchdog itself |
| Slack incoming webhook + interaction endpoint | Alert delivery and "mark as handled" button |
| Resend or postmark | Email delivery for backup alerts |

All credentials live in Vercel environment variables. No secrets in the repo.

The choice of Next.js over a smaller framework is because the dashboard, API routes, cron handlers, and Slack interaction endpoints all live in one deploy unit. Splitting them across multiple services costs more than it saves at this scale.

## Phasing

| Phase | Scope | Estimate |
|---|---|---|
| **v2.0** | Welcome email verification only. Schedule registry, verifier, alert router (Slack + email), basic dashboard, heartbeat. Pull-only from email tool. | 2-3 days |
| **v2.1** | Add calendar invite and Slack channel access verification. | 1 day |
| **v2.2** | Add session reminder and weekly materials drop verification. Adds the recurring-comm pattern. | 1-2 days |
| **v2.3** | Push integration (webhooks from email tool, if supported) as an additive signal. Detect drift between push and pull. | 1 day |
| **v2.4** | Google Sheet schedule editor for ops. | 1 day |

v2.0 alone catches the Cohort 12 failure pattern. Everything after deepens coverage; nothing after is required to make the initial fix viable.

## Definition of done for v2.0

1. Schedule registry parses both YAML and a designated Google Sheet
2. Verifier runs every 15 minutes via Vercel Cron
3. For each cohort welcome email in the round, verifier runs the five checks and emits a result
4. Failures alert to `#ops-watchdog` Slack channel and Course Ops Lead email within one verifier cycle (max 30 minutes after scheduled send time)
5. Alert dedup prevents repeat alerts within the same failure
6. Escalation triggers at 1h and 4h
7. `/status` dashboard shows last 14 days
8. Heartbeat external cron is configured and pages on watchdog downtime
9. Integration endpoint `/api/verification` returns a verification record for the v1 checklist tool to consume
10. Test mode: a `DRY_RUN=true` env var makes the verifier read from a fixture file instead of the live email tool, and routes alerts to a `#ops-watchdog-test` channel
11. README documents how to add a new round to the schedule, how to test locally, and how to silence a known false positive

## Failure modes the design has to handle

| Failure | Handling |
|---|---|
| Email tool API is down at check time | Verifier marks result as `unknown` (not `fail`), retries on next tick, alerts only if `unknown` persists for >2 hours |
| Email tool rate-limits the watchdog | Verifier backs off exponentially, caches last successful result per (cohort, comm) for up to 1 hour |
| Watchdog itself is down | External heartbeat cron alerts the Course Ops Lead within 5 minutes |
| Schedule has a typo and a comm is missing from the registry | Verifier cannot detect a missing comm if it doesn't know to expect it. Mitigation: weekly schedule audit cron compares the schedule registry to the roster (every cohort with a future session 1 must have all five comm types scheduled) |
| Roster changes between check runs (someone admitted on Friday) | Verifier always reloads roster on every run; uses the latest |
| Cohort 12-style failure repeats: automation tool silently fails to send for an entire cohort | Caught by the `send_complete` check within 30 minutes |

## Privacy and security considerations

- Participant emails will pass through verifier memory and KV. Encrypt at rest where the platform supports it (Upstash supports encryption in transit and at rest by default). No PII written to logs at info level.
- API tokens for the email tool, Slack, and Google Calendar are stored as Vercel encrypted environment variables. Rotation cadence: every 90 days. Document in README.
- Status dashboard requires a shared static token or Vercel password protection. Not public.
- Audit log of every alert and acknowledgement is appended to Vercel Blob with a 12-month retention.

## Open questions for review

1. **Which email automation tool's API are we integrating against first?** v2.0 has to pick one. Recommend asking Li-Lian which tool the welcome email automation actually runs on. Adapter pattern means adding a second tool later is small.
2. **Slack workspace and incoming webhook permissions** - needs ops approval to create `#ops-watchdog` channel and install the webhook
3. **On-call rotation** - does the alert page Course Ops Lead 24/7, or only during UK business hours? Recommend business hours for v2.0 and add follow-the-sun later if needed
4. **Backup alert path for the Course Ops Lead** - if Slack and email both fail, do we add SMS via Twilio? Recommend deferring to v2.3 unless a real incident motivates it

None of these block committing to the direction.

## Alternatives considered

### Build the verification inside the v1 HTML file

Bring evidence ingestion into the browser. **Rejected** because the failure being fixed is "nobody noticed". A tool that requires a human to open it is not a watchdog. The watchdog has to run on its own.

### Wrap the email automation tool itself with a "did this send work" feature

Sounds tempting. **Rejected** because the failure was inside the email tool. Wrapping the failing system with more of itself does not add independence. The watchdog must be a different system in a different runtime.

### A Zapier-based "if cohort X welcome email is supposed to have sent by Y, and it has not, ping Slack" automation

Lowest build cost. **Rejected** because Zapier is itself a possible point of failure (if Zapier is the automation tool that failed for Cohort 12, doubling down on Zapier is the wrong shape). The watchdog has to live outside the systems it watches.

### A weekly "did everything launch correctly" report

Too late. The Cohort 12 participants emailed asking where their welcome email was. A weekly report would discover that mid-week, after the damage. The watchdog has to be real-time enough that ops can recover before participants notice.

## Next steps

1. User reviews this spec
2. On approval, invoke `superpowers:writing-plans` to produce the v2.0 implementation plan
3. Plan covers: Vercel project init, schedule registry parser, first adapter (whichever email tool), verifier cron handler, alert router, basic dashboard, integration endpoint, heartbeat setup, tests
4. Implementation starts only after the plan is approved
