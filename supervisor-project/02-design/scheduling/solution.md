# Commander — Scheduling Solution

This is the reference for how Commander decides when each CAMT report runs and what time
period it covers.

> Visual: `scheduling-flow.drawio` (draw.io / diagrams.net) — trigger sources, the scheduled
> job lifecycle, the three window shapes on a timeline, and clustered Quartz across pods.

> Building or reviewing the code? `implementation-reference.md` has the exact config schema,
> generated cron expressions, admin-endpoint payloads, and edge cases this document leaves out
> on purpose, to stay readable.

---

## 1. What scheduling is responsible for

Scheduling has exactly one job: on a timetable, wake up and say *"produce this report type,
covering this time window."* It then hands that instruction off to the message-production
pipeline and steps away — it doesn't build the report, resolve any payment-type data (accounts
or aliases), or track whether the run succeeded. Those are separate concerns, documented in
`../message-pipeline/` and `../data-retrieval/` respectively.

Every firing boils down to five facts, handed to the pipeline as one package:

**report type · frequency · the moment it was scheduled for · window start · window end**

---

## 2. The three ways a report's window is calculated

Every report frequency falls into one of three shapes — this is the whole mental model:

- **Rolling — "the last N minutes."** The window is a fixed look-back ending at the fire
  time. A run every 30 minutes at 13:00 covers 12:30–13:00; at 13:30 it covers 13:00–13:30.
  Used for the highest-frequency intraday reports.
- **Boundary — "since the previous checkpoint today."** The day has a fixed set of clock
  times; each run covers the stretch from the previous one to this one, with midnight as an
  implicit first checkpoint. Boundaries 10:00, 13:00, 18:00, 21:00 mean the 10:00 run covers
  00:00–10:00 (it's the first of the day), the 13:00 run covers 10:00–13:00, and so on. Used
  for the "N times a day" notification reports, and also for the every-2-hour / every-4-hour
  intraday reports — anchored to midnight the same way.
- **Calendar-day — "all of yesterday."** The window is the entire previous calendar day,
  midnight to midnight; the run just happens the next morning to deliver it. A 06:00 run
  covers the whole day before, 00:00–24:00. Used for the end-of-day statements.

All window math happens in one configured business timezone, **`Europe/Stockholm`**, then
converts to absolute time for the outgoing message.

---

## 3. Every report, at a glance

| Report type(s) | Frequency | Shape | Fires at | Days | Reporting window |
|---|---|---|---|---|---|
| CAMT052B, CAMT052BT | Every 30 min | Rolling | 00:30 → 21:00 | Mon–Fri | last 30 min |
| CAMT052B, CAMT052BT | Every hour | Rolling | 01:00 → 21:00 | Mon–Fri | last 60 min |
| CAMT052B, CAMT052BT | Every 2 h | Boundary | 03:00, 05:00 … 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–03:00 |
| CAMT052B, CAMT052BT | Every 4 h | Boundary | 05:00, 09:00, 13:00, 17:00, 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–05:00 |
| CAMT054C | Once a day | Boundary | 21:00 | Mon–Fri | 00:00–21:00, same day |
| CAMT054C | 4× a day | Boundary | 10:00, 13:00, 18:00, 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–10:00 |
| CAMT054C | 8× a day | Boundary | 03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00 | Mon–Fri | since previous checkpoint; first run covers 00:00–03:00 |
| CAMT053S, CAMT053E, CAMT054D | End of day | Calendar-day | 06:00 | Tue–Sat | the whole previous calendar day |
| any of the above | Never | — | not scheduled | — | — |

A few notes:

- **21:00 → 24:00 is never reported same-day** by any rolling or boundary frequency — their
  last checkpoint is 21:00.
- **Every-2-hours / every-4-hours anchor to midnight**, not to their first fire time. This is a
  deliberate difference from the legacy system, which treats these two as a rolling look-back
  and leaves 00:00–01:00 unreported each day — worth flagging during cutover comparisons.
- **`Never`** is the marker for configurations that are recipients of the external
  balance-push flow, which is triggered by an inbound message, not a timetable. No schedule
  ever picks a `Never` config up.
- **Everything in this table is configuration, not fixed code** — the fire times, the
  boundaries, the days. Changing any of it means a redeploy, not a live edit (§5).

---

## 4. What happens when a schedule fires

1. The timetable is due. Because the scheduler runs in **clustered mode**, exactly **one** pod
   across the whole deployment picks up the firing — this is a property of the clustering
   technology, not something the application has to coordinate itself.
2. That pod resolves the window using the shape rules above, from the time the firing was
   **scheduled for** — never the wall-clock time it happened to wake up at. For a normal firing
   these are the same instant; the distinction matters for a manual backfill or a recovered
   run, where the report must cover the period it was meant to, not whenever it actually ran.
3. It creates one tracking record (a "Run") as its **first durable action** — precisely so
   that anything that goes wrong afterward leaves a trace recovery can act on (§8).
4. It checks a **feature flag for that report type** — is this job allowed to run at all. If
   the flag is off, the Run is marked skipped right there and nothing is handed to the
   pipeline — no configurations are resolved, nothing is built. If the flag is on, it hands the
   pipeline the five facts from §1. The pipeline takes it from there — finding the matching
   configurations, resolving their data, building and publishing the messages.

   A skipped Run still occupies that slot — that's final, not something to backfill (§6). If
   that window's report is still wanted, that's an on-demand request instead, not a retry.

**Daylight saving.** Clocks skip an hour forward each spring and repeat an hour each autumn. A
fire time that falls in the skipped stretch shifts forward to the next real moment; one that
falls in the repeated stretch fires once, at its first occurrence — not again the second time
around.

For example, a 02:30 boundary: in spring, clocks jump straight from 02:00 to 03:00, so 02:30
never happens — that boundary fires at 03:00 instead. In autumn, clocks fall back from 03:00 to
02:00, so 02:30 happens twice — the boundary only fires on the first pass. (No Boundary
checkpoint sits in 02:00–03:00 today, so this hasn't come up in practice yet — but the rule is
ready for whenever one does.)

**Rolling frequencies hit this every year** — `EVERY_1_HOUR` fires at 02:00, `EVERY_30_MIN` at
02:00 and 02:30, both inside that stretch. The same rule applies, but with a different result:
a rolling window is just "fire time minus the interval," so the fire next to the transition
ends up with a window that starts and ends at the same instant — an empty report, not a wrong
one.

| Frequency | Fires inside 02:00–03:00 | Spring (hour is skipped) | Autumn (hour repeats) |
|---|---|---|---|
| `EVERY_1_HOUR` | 02:00 | Doesn't happen — the next fire is the regular 03:00, whose window collapses to zero length | Fires once, at the first pass; the regular 03:00 fire picks up from there |
| `EVERY_30_MIN` | 02:00, 02:30 | Neither happens — same as above | Each fires once, at its first pass |

Either way: one report comes out empty, and one hour of real activity around the transition
simply isn't captured that day. Nothing catches it automatically; backfill it explicitly if
it's ever needed.

A window's actual length can also shift by an hour on a transition day, even with no boundary
inside the gap or overlap — take the end-of-day window, normally a full 24 hours. On the spring
day it's only 23 hours, because the day it covers lost an hour to the clock jump; on the autumn
day it's 25 hours, because that day gained an hour back. Nothing corrects for this — the window
is simply "midnight to midnight" by the clock, whatever that comes out to in elapsed time that
day. Expected, not a bug.

---

## 5. How a schedule is set up

Schedules are **fixed configuration, changed by redeploy** — there is no live admin UI for
editing timetables, and that's a deliberate boundary: an operator can pause, resume, or run a
schedule out of band (§7), but reshaping *what* a timetable looks like is a config change.

A schedule is described in plain terms — which report types, what frequency, which days, and
either an interval or an explicit list of boundary times — and the application **generates**
the underlying timer expression from that, rather than the operator hand-writing both. This
matters because the boundary list already drives the window calculation; if the timer were
also hand-written, the two could silently drift apart. A single schedule entry can cover
several report types that share a timetable (the three end-of-day reports, for instance); the
application expands that into one independent schedule per report type at startup.

One example per shape:

```properties
commander.scheduling.timezone = Europe/Stockholm

# Rolling — CAMT052B/BT, hourly
commander.scheduling.triggers[0].report-types = CAMT052B, CAMT052BT
commander.scheduling.triggers[0].frequency     = EVERY_1_HOUR
commander.scheduling.triggers[0].days          = MON-FRI
commander.scheduling.triggers[0].shape         = ROLLING
commander.scheduling.triggers[0].interval      = { first: 01:00, step: 1h, last: 21:00 }

# Boundary — CAMT054C, eight times a day
commander.scheduling.triggers[1].report-types = CAMT054C
commander.scheduling.triggers[1].frequency     = EIGHT_TIMES_PER_DAY
commander.scheduling.triggers[1].days          = MON-FRI
commander.scheduling.triggers[1].shape         = BOUNDARY
commander.scheduling.triggers[1].boundaries    = 03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00

# Calendar-day — the end-of-day statements
commander.scheduling.triggers[2].report-types = CAMT053S, CAMT053E, CAMT054D
commander.scheduling.triggers[2].frequency     = END_OF_DAY
commander.scheduling.triggers[2].days          = TUE-SAT
commander.scheduling.triggers[2].shape         = CALENDAR_DAY
commander.scheduling.triggers[2].fire-at       = 06:00
```

A raw timer override exists for test environments, so tests can fire on a fast cadence instead
of waiting for real clock time — it's not available in production. For example, overriding the
eight-times-a-day schedule above to fire every 2 minutes instead of at its real boundary times:

```properties
commander.scheduling.triggers[1].cron-override = 0 0/2 * ? * MON-FRI
```

The reporting window still follows the real boundary rules above — only *when the job wakes
up* changes, so a test run gets a full pass through the pipeline every couple of minutes
without waiting for the actual clock.

---

## 6. When a firing is missed

A firing can go missing for more reasons than just an outage — the whole system being down is
the most common, but it also happens if an operator has deliberately paused that schedule (for
example, to protect production during an environment issue). Whatever the cause, it's handled
the same way, and an operator always has the same option regardless of why a slot was missed:
trigger it explicitly.

**The automatic policy is: do nothing.** The missed slot is skipped entirely; the timetable
simply resumes at its next natural time and produces that window normally. Firings paused over
aren't queued up and released in a burst on resume — they're just gone, the same as if the
system had been down for that stretch. There is no automatic catch-up, ever, for any report —
for a missed end-of-day, where nothing else will produce that day's statement, an alert is
recommended so an operator knows to act.

Recovering a missed slot is always a deliberate action, and it's available for **any** missed
slot, sub-daily or end-of-day alike — there's no restriction on which ones can be backfilled.
An operator (or a small admin endpoint) fires the schedule again, passing the missed slot's
time, and it produces exactly the window that firing would have produced on time. This is
idempotent — re-issuing a backfill for a slot that already ran is a harmless no-op.

This is also the intended shape of a deliberate pause: pause the schedule to ride out the
issue, fix it, resume for firings going forward, then — once it's agreed which slots actually
matter — backfill exactly those.

---

## 7. Admin controls

One authenticated, admin-only surface, all keyed on `(report type, frequency)`:

| Action | What it does |
|---|---|
| **Run now** | Fires a schedule immediately, out of band from its timetable. Optionally targets a specific slot's time, producing that slot's window rather than "now". |
| **Backfill** | Recovers one or more missed slots — an explicit list, or a time range expanded into that frequency's checkpoints. Fires one run per slot; safe to re-issue. |
| **Pause** | Stops a schedule from firing, cluster-wide, until resumed. Survives a redeploy as long as the schedule still exists in config. |
| **Resume** | Restarts a paused schedule going forward. Does **not** backfill what was missed while paused — use Backfill for that. |
| **Status** | Lists every schedule: paused or active, last fire time, next fire time. |

---

## 8. A few reliability details worth knowing

- **Overlapping runs are allowed on purpose.** If a 12:30–13:00 run is still going when the
  13:00–13:30 run fires, both run at the same time and both publish — a slow run must never
  hold back the next window's report.
- **A repeated firing for the same slot is a harmless no-op, not an error.** This can happen if
  Quartz fires a trigger twice for what's effectively the same instant, or if a manual backfill
  happens to target a slot that also fired normally around the same time. The tracking record
  created for a slot is unique to that exact (report type, frequency, scheduled time)
  combination — so whichever firing gets there first creates it, and any other firing for the
  exact same slot finds it already exists, recognizes the duplicate, and exits without doing
  any further work. Nothing is built or published twice, and it isn't treated as a failure.
- **A crashed pod's work is always recovered — which mechanism handles it just depends on
  timing.**
    - **Pod dies in the instant before it could even create that record:** the scheduler's own
      clustering re-fires that one firing on a live pod, and it starts fresh — since, as far as
      anything can tell, that firing never actually began.
    - **Pod dies after the tracking record was created, but before the run finished:** the
      pipeline's own recovery mechanism notices (it watches for runs that have gone quiet) and
      resumes from where the pod stopped.

  These two never clash, because the re-fired attempt always does the same first check: try to
  create the tracking record. If it doesn't exist yet, create it and carry on normally. If it
  already exists — meaning the original pod actually got further than expected — just notice
  that and stop (the same harmless no-op as a repeated firing, above). A re-fired attempt never
  tries to pick up half-finished work itself; only the pipeline's own recovery mechanism does
  that. Either way, nothing is silently lost.
- **Redeploys clean up after themselves.** If a previous configuration's timetables are still
  registered, the application removes them at startup, before the scheduler starts running —
  so nothing fires against a stale schedule.

---

## 9. What to verify before this goes live

Not open design questions — a short checklist to confirm once this is implemented, under a
real clustered deployment:

- The do-nothing policy (§6) skips a genuinely missed firing cleanly, and the next scheduled
  fire is unaffected.
- The same holds for a trigger resumed after being paused across one or more of its fire times.
- When a pod is killed and the scheduler's clustering re-fires the affected job (§8): if no
  tracking record existed yet, one is created and processing proceeds normally; if one already
  existed, the re-fired execution recognizes it and exits cleanly — no duplicate record, no
  double-published report either way.
