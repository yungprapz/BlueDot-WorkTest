# Reminders and notifications: from passive tools to a proactive system

Status: draft, awaiting review
Author: Prapthi Agarwala (with Claude)
Date: 2026-05-26
Companion to: bluedot-launch-checklist.html (v1) and the comms watchdog (v2 spec)

## What this is

The v1 checklist sits quietly until a human opens it. The v2 watchdog sends alerts to a Slack channel and an email inbox, both of which require somebody to be watching them.

This spec adds the layer between those two: a notifications system that actively reaches the right person at the right time, on a channel they will actually see, so that "we have a tool" and "we noticed in time" stop being two different sentences.

Three signals trigger a notification:

1. **A deadline is approaching and the checklist is not complete** (predictable, schedule-driven)
2. **The watchdog has detected a failure** (reactive, event-driven)
3. **A scheduled comm's fire time has passed and no evidence of it firing has arrived yet** (predictive, time-and-event)

Three channels carry them:

1. **In-app banner** when the checklist tool is open
2. **Browser notification** when the browser is open but the tool tab is in the background
3. **Email or SMS** when the browser is closed

The point is to remove the assumption that ops will happen to look at the right system at the right time.

## Why we need this even with the watchdog

The watchdog alerts the Slack channel and the Course Ops Lead inbox. Both work, until they don't:

- The Slack channel might be muted on a Saturday morning
- The Course Ops Lead might be in back-to-back meetings or on annual leave
- The escalation chain takes time and assumes someone is at a keyboard

This layer is about the last mile. It targets a specific human (the on-call Course Ops Lead), on a channel that interrupts (browser notification, SMS), at a moment that matters (right before launch, or immediately after a missed fire). It does not replace Slack and email; it adds a final, harder-to-ignore path.

## Goals

1. Surface incomplete checklist work proactively as the session date approaches
2. Surface watchdog failures inside the checklist tool so the human is looking at the right view when they open it
3. Wake the human up via browser notification if the tool is not in focus
4. Fall back to email or SMS if the browser is closed and the situation is serious enough
5. Be quiet otherwise. No alert fatigue. Every notification should be one the recipient is glad they got.

Non-goals:

- Replacing Slack as the team's primary alert channel
- Phone calls or human paging chains
- Notifications for non-launch concerns (facilitator-side comms, participant-side comms)
- A separate mobile app

## What the user actually sees

### Reminder, deadline-driven

> **Round 8 launches in 24 hours**
> Cohort 12 has 7 unchecked items, including 3 in Welcome email and 2 in Slack channel.
> [Open checklist] [Snooze 2 hours] [Dismiss]

Triggered T-72h, T-24h, T-6h before each cohort's session 1 if its checklist progress is below 90%. Higher priority means louder channel (banner -> browser notification -> email).

### Notification, watchdog-driven failure

> **Watchdog: welcome email failed for Cohort 12**
> 5 of 12 recipients missing from delivery log. Scheduled send was 2 hours ago.
> [See evidence] [Mark as handled]

Triggered immediately when the watchdog emits a `failed` verification result for any cohort. Surfaces inside the tool as a red banner pinned to the top of every view, plus a browser notification, plus email if the situation persists past 1 hour.

### Notification, predictive (fire time passed, no evidence)

> **Welcome email for Cohort 8 was scheduled 30 minutes ago, no evidence found yet**
> The watchdog has not yet confirmed delivery. This may resolve on the next check.
> [Open watchdog status] [Snooze 30 minutes]

Triggered when the watchdog returns `unknown` for an expected send for more than one cycle. Quieter than a `failed`, because it might be a transient API issue.

## Architecture

### Component map

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│  Checklist tool (v1 + v3)   │     │  Comms watchdog (v2)        │
│                             │     │                             │
│  ┌───────────────────────┐  │     │  ┌───────────────────────┐  │
│  │ Reminder scheduler    │  │     │  │ Verifier              │  │
│  │ (browser, in-tab)     │  │     │  │                       │  │
│  │ - reads session dates │  │     │  │ Emits results per     │  │
│  │ - schedules timers    │  │     │  │ cohort x comm         │  │
│  │ - fires at T-72/24/6h │  │     │  └─────────┬─────────────┘  │
│  └───────────┬───────────┘  │     │            │                │
│              │              │     │            ▼                │
│  ┌───────────▼───────────┐  │     │  ┌───────────────────────┐  │
│  │ Notification router   │◀─┼─────┼──┤ Alert router          │  │
│  │ - in-app banner       │  │ poll │  │ Slack + email +       │  │
│  │ - browser Notification│  │      │  │ watchdog-push channel │  │
│  │ - Service Worker push │  │      │  └───────────────────────┘  │
│  └───────────────────────┘  │     │                             │
└─────────────────────────────┘     └─────────────────────────────┘
                ▲                                  │
                │                                  ▼
                │                    ┌─────────────────────────────┐
                │                    │  Push gateway (optional)    │
                │                    │  - Web Push protocol        │
                │                    │  - Twilio for SMS fallback  │
                │                    └─────────────────────────────┘
                │
        Subscription stored in
        user preferences (browser localStorage)
```

### Two delivery modes, picked by environment

**Mode A: tool is open in a foreground tab**
- Reminder scheduler ticks on `setTimeout` against the session dates the user entered at setup
- Watchdog data fetched via polling the `/api/verification` endpoint every 60 seconds
- All notifications render as in-app banners at the top of the tool

**Mode B: tool is in a background tab or browser is open elsewhere**
- A Service Worker registered by the tool runs reminders even when the tab is not focused
- The Service Worker keeps a local schedule of upcoming reminder times (derived from the round's session dates)
- For watchdog events, the Service Worker subscribes to Web Push from the watchdog. Watchdog sends a push when it emits a failure
- Browser shows a native Notification

**Mode C: browser is closed**
- Service Workers cannot run when the browser is closed
- The watchdog (existing v2 component) sends the user an email or SMS directly, bypassing the browser
- This is the watchdog's existing alert path. The reminder scheduler stops contributing in Mode C; only watchdog-driven notifications survive
- The user enrolls their email/phone in the tool's settings, which calls the watchdog `/api/subscribe` endpoint to register

The shift between modes is invisible to the user. They just see the alert.

### Why a Service Worker

Without a Service Worker, browser-side reminders only fire while the tab is in the foreground. Ops people close tabs. A Service Worker registered by the tool gives us reliable background scheduling for the predictable reminders, and a subscription endpoint for the watchdog to push to.

The Service Worker is small and idempotent. It loads a static schedule (derived from the round's session dates at registration), wakes at the scheduled times, and posts a notification. Watchdog push events arrive as `push` events handled by the same worker.

The Service Worker stops working when the browser is fully closed or the OS terminates it. That is the gap email or SMS covers in Mode C.

### Why Web Push, not WebSockets

A WebSocket would require the tool to be open and maintain a connection. Web Push works through the Service Worker even when the tab is not focused, which is the whole point. Web Push needs a small relay endpoint on the watchdog (using the `web-push` library); the watchdog already runs as a Vercel service, so this is one more API route.

## Trigger rules

### Reminder rules (predictable, schedule-driven)

| Trigger | Condition | Channel |
|---|---|---|
| T-72h before any cohort's session 1 | That cohort's checklist completion < 90% | In-app banner if open, browser notification otherwise |
| T-24h before any cohort's session 1 | Same | In-app banner, browser notification, and watchdog email if the cohort completion is < 70% |
| T-6h before any cohort's session N | That cohort's tech check items (Section 7) < 100% | In-app banner, browser notification |
| T-6h before round end | Any unresolved issue in the issue log | In-app banner only |

Reminders are dismissible (suppress this exact trigger for this round) or snoozeable (re-fire in 2 hours).

### Watchdog-driven rules (reactive)

| Watchdog result | Channel | Notes |
|---|---|---|
| `failed` for any cohort comm | All three modes. Banner pinned to top of every view. Browser notification. Watchdog also sends Slack + email (v2 behavior). | This is the loudest. Cannot be snoozed, only marked as handled. |
| `unknown` for >2 hours on a comm whose fire time has passed | In-app banner if open. Browser notification only if the user has opted into "tentative alerts" in settings. | Quieter because transients exist. |
| `unknown` for >6 hours | Promote to same severity as `failed` | Catches the case where the email tool's API has been down for a long time |

### Predictive rules (time-driven, watchdog-aware)

If the local schedule says a comm should have fired in the last 30 minutes, but the latest watchdog verification result is still `unknown` (not yet observed), the tool surfaces a banner: "Welcome email for Cohort N was scheduled at T-0:30, no evidence yet, watchdog will check again in N minutes."

This is the gentlest of the three. It is informational, not action-requiring. It exists because seeing the gap close in real time is more reassuring than waiting for a binary `pass` or `failed`.

## What gets stored where

In the tool (browser localStorage):

- Reminder schedule for the active round (derived from session dates)
- Snooze and dismiss state per trigger
- User notification preferences (channels enabled, quiet hours)
- Service Worker registration state

In the watchdog (Vercel KV):

- Web Push subscriptions (the tool registers, watchdog pushes)
- Per-user notification preferences (email address, phone number, opted-in channels)
- Dedup state for `failed` and `unknown` events (already in v2 spec)

The tool never sends participant data anywhere new. Only "I, the ops user, want to receive notifications at this email or phone" leaves the browser, and only when the user enrolls explicitly.

## User preferences and quiet hours

A new settings panel in the tool covers:

- Channels: browser notifications (on by default if permission granted), email (off by default), SMS (off by default)
- Quiet hours: a per-day time range when only critical notifications fire. Critical = watchdog `failed` only.
- Recipient: email address and optional phone number
- Stop bothering me: master kill switch that disables everything for 24 hours

The settings panel is also where the user grants browser Notification permission. The flow is:

1. First time a reminder would fire, tool shows: "Do you want to enable browser notifications for this checklist?"
2. If yes, browser prompt appears
3. If granted, Service Worker registers, watchdog push subscription is created
4. If denied or dismissed, fall back to in-app only

No notification surface ever asks again after a denial. The user can re-enable from settings.

## Phasing

| Phase | Scope | Estimate |
|---|---|---|
| **v3.0** | In-app banners only. No Service Worker, no browser notifications. Reminders fire when the tool tab is focused. Watchdog polling shows banners for `failed` and `unknown`. | 1 day |
| **v3.1** | Service Worker for foreground + background reminders. Browser Notification API. Web Push subscription with the watchdog. | 2 days |
| **v3.2** | Email and SMS fallback via watchdog. Settings panel for recipient info and quiet hours. | 1-2 days |
| **v3.3** | "Stop bothering me" preferences, snooze, dismiss-this-trigger persistence. Tuning. | 1 day |

v3.0 is shippable independently and delivers most of the value if the user habitually keeps the tab open. Each phase strictly adds; nothing in an earlier phase breaks.

## Definition of done for v3.0

1. Reminder triggers fire at T-72h, T-24h, T-6h relative to entered session dates, surfaced as banners at the top of any view in the tool
2. Polling fetches the watchdog `/api/verification` endpoint every 60 seconds when the tool is open
3. Any `failed` result becomes a red banner pinned to all views with the failure detail and a "Mark as handled" button
4. Any `unknown` over 2 hours becomes an amber banner
5. Snooze and dismiss buttons work and persist for the active round
6. If the watchdog is unreachable, banners silently degrade off; no error spam
7. The existing v1 functionality is unchanged for users who never see a reminder or a watchdog signal

## Risks and how the design handles them

| Risk | Mitigation |
|---|---|
| Notification fatigue, users disable everything | Tiered severity from the start. Reminders snoozeable. Quiet hours respected. No notifications for cohorts already at 100%. Critical-only fallback in settings. |
| Service Worker bugs causing missed reminders | Each reminder also fires from the foreground scheduler when the tab is focused, so a SW failure does not lose reminders for users who keep the tool open. Email fallback covers the rest. |
| Permission denied for browser notifications | Falls back to in-app banners. Settings panel can re-prompt only when user explicitly clicks "enable". |
| User has the tool open in two tabs, gets each notification twice | Service Worker scope deduplicates by trigger ID. Foreground banner is per-tab but only one tab will be focused at a time. |
| Watchdog push spoofing | Web Push subscriptions are signed; the watchdog uses VAPID keys; tool verifies subscription origin. |
| User leaves BlueDot, watchdog still has their email subscribed | Settings panel has an "unsubscribe everywhere" button. Quarterly cron in the watchdog prunes subscriptions inactive > 90 days. |
| Mobile / OS-level "do not disturb" mode | Browser notifications respect OS DND. Users on the move stay opted into SMS, which OS DND does not silently swallow. |

## Privacy considerations

- Browser notification permission is requested explicitly, with a tool-side explanation before the browser prompt
- Email and SMS recipients are stored in the watchdog KV, encrypted at rest. Not shared.
- Subscription metadata visible to the watchdog: just the user's enrollment record (round number, email, phone, channels, quiet hours). No checklist contents leave the browser.
- The Service Worker scope is the tool's path only; it cannot intercept other browser activity
- All preference changes are auditable via the watchdog's audit log (v2 spec adds this)

## Alternatives considered

### Use only Slack and email, do not build the third path

Lowest cost. Rejected because the original Cohort 12 failure was discovered by participants, not by us, even though we have a Slack channel and an inbox. Existing channels are necessary but not sufficient.

### Build a mobile app

Tempting because mobile push is the most reliable notification channel. Rejected because the build and ongoing maintenance cost for a small ops team is prohibitive. Web Push covers ~85% of the same value at a fraction of the cost.

### Integrate with a paging service (PagerDuty, Opsgenie)

Right answer when on-call rotation is mature. Rejected as premature: BlueDot's ops team is small and a single person likely owns the launch each round. Direct email and SMS from the watchdog covers the use case until rotation complexity warrants more.

### Put all reminder logic in the watchdog and let the watchdog email constantly

The watchdog already does email and Slack on failure. We could add scheduled email reminders too. Rejected because deadline-driven reminders depend on local state the watchdog does not own (which checklist items are unchecked). Pushing that state to the watchdog would expand the watchdog's responsibilities and make it stateful in a way that adds maintenance burden. Letting the tool own deadline reminders keeps each component narrow.

### Use only the browser's built-in scheduled notifications API

Modern browsers expose a `showTrigger` API for scheduled notifications. Rejected because it is Chromium-only and still flagged experimental. Service Worker timers are universal enough.

## Open questions for review

1. **Does the v1 tool need to expose session times per session, or just per round?** Today it captures session 1 only. T-6h-per-session reminders require knowing each week's session datetime per cohort. Recommend adding optional session schedule to setup.
2. **Default quiet hours.** Recommend 21:00 to 07:00 in the user's local time. UK ops can adjust.
3. **SMS provider.** Twilio is the easy default. Cost is ~$0.01 per SMS, manageable at expected volume. Confirm with Li-Lian.
4. **Notification copy review.** A short pass on the recommended text strings to make sure they match BlueDot voice.

None of these block committing to the design.

## How this composes with v1 and v2

| Layer | What it owns | When it acts |
|---|---|---|
| v1 launch checklist | The human checks, the issue log, the failure response table | When a human opens it |
| v2 comms watchdog | Verifying scheduled sends, alerting Slack and email | Continuously, autonomously |
| v3 reminders and notifications | Reaching the right human at the right time on the right channel | Predictively (deadlines) and reactively (watchdog signals) |

The three are independent shippable units. v1 is shipped. v2 is specified. v3 builds on top of both but degrades gracefully if either is missing: without the watchdog, v3.0 still does deadline reminders; without v3, v1 and v2 both still work.

## Next steps

1. User reviews this spec
2. On approval, invoke `superpowers:writing-plans` to produce the v3.0 implementation plan (in-app banners only is the smallest first slice)
3. v3.0 plan covers: banner component, deadline scheduler, watchdog polling integration, snooze and dismiss persistence, tests
4. Implementation starts only after the plan is approved
