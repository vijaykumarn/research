# Commander — Scheduling, In Plain Terms

A conceptual overview of how Commander decides **when** each report runs and **what time
period** each run covers. No code, no schema. The detailed design is in
`solution_v01.md`; the message-generation pipeline it hands off to is in
`../message-pipeline/how-it-works.md`.

---

## 1. What "scheduling" is responsible for

Commander produces CAMT report messages on a timetable. The scheduling part has exactly one
job:

> On a timetable, wake up and say: *"produce **this report type**, covering **this time
> window**."*

It then hands that instruction to the message-generation pipeline and steps away. It does not
build the messages, talk to the database beyond triggering, or track whether the run
succeeded — that's all the pipeline.

So every firing produces just five facts:

**report type · frequency · the moment it was scheduled for · window start · window end**

---

## 2. The two questions every report answers

**"How often does this report run?"** — its *frequency*. Each report configuration carries one
frequency value (every 30 minutes, hourly, four times a day, once a day at end of day, …).

**"What period of activity does one run cover?"** — its *reporting window*. This is the part
that needs care: a report that runs at 13:00 isn't reporting *about* 13:00, it's reporting
about some stretch of time leading up to it. Which stretch depends on the report.

---

## 3. The core idea: three schedule "shapes"

Every frequency falls into one of three shapes. This is the whole mental model.

### Shape 1 — Rolling ("the last N minutes")

The window is simply a fixed look-back ending at the fire time.

- Runs every 30 minutes → each run covers the **previous 30 minutes**.
- 13:00 run → window 12:30–13:00. 13:30 run → 13:00–13:30. And so on.

Used for the high-frequency intraday reports (`EVERY_30_MIN`, `EVERY_1_HOUR`).

### Shape 2 — Boundary ("since the previous checkpoint today")

The day has a set of fixed clock times — *boundaries* — and each run covers the stretch from
the previous boundary to this one. Midnight is always an implicit first boundary, so the
first run of the day reaches back to 00:00.

- Boundaries 10:00, 13:00, 18:00, 21:00.
- 10:00 run → covers **00:00–10:00** (from midnight, since it's the first).
- 13:00 run → covers 10:00–13:00.
- 21:00 run → covers 18:00–21:00.

Used for the "N times a day" notification reports (`FOUR_TIMES_PER_DAY`,
`EIGHT_TIMES_PER_DAY`, `ONCE_PER_DAY`) and the every-2-hours / every-4-hours intraday reports —
so their first run of the day covers from midnight (00:00–03:00 / 00:00–05:00), not from an
hour in.

### Shape 3 — Calendar-day ("all of yesterday")

The window is the entire previous calendar day, midnight to midnight. The run just happens the
next morning to deliver it.

- Runs at 06:00 → covers **the whole of the day before**, 00:00 to 24:00.

Used for the end-of-day statements and notifications (`END_OF_DAY`: CAMT053S, CAMT053E,
CAMT054D).

---

## 4. Which report uses which shape

| Report type(s) | Frequency | Shape | Runs | Covers |
|---|---|---|---|---|
| CAMT052B, CAMT052BT | Every 30 min / hourly | Rolling | 00:30–21:00, weekdays | the last 30 min / 60 min |
| CAMT052B, CAMT052BT | Every 2 h / every 4 h | Boundary | from 03:00 / 05:00 to 21:00, weekdays | since the previous checkpoint; first run of the day covers from midnight |
| CAMT054C | Once a day | Boundary | 21:00, weekdays | 00:00–21:00 that day |
| CAMT054C | 4× / 8× a day | Boundary | fixed times ending 21:00, weekdays | since the previous checkpoint |
| CAMT053S, CAMT053E, CAMT054D | End of day | Calendar-day | 06:00, Tue–Sat | the whole previous calendar day |

Three housekeeping points:

- **Timezone.** Every time above is in one configured business timezone. All the window maths
  happens in that zone, then converts to absolute time for the message.
- **The "never" marker.** Some report configurations exist only for the external balance-push
  (PHT) flow and must never be picked up by the scheduler. They carry a special frequency
  value (`NEVER`) that no schedule matches, so they're excluded automatically — no special
  code.
- **A behaviour difference from the legacy system, worth flagging for cutover.** The system
  being replaced treats every-2-hours / every-4-hours as a rolling look-back, so its first run
  of the day only reaches back to 01:00 — 00:00–01:00 goes unreported. Commander closes that
  gap by anchoring these two to midnight, same as the other boundary frequencies.

---

## 5. How a schedule is set up

Schedules are **fixed configuration, changed by redeploy** — there is no admin UI for editing
timetables, and that's deliberate scope-limiting.

The important conceptual choice: an operator describes a schedule in plain terms — *"these
report types, this frequency, these days, these boundary times"* — and the application
**generates the low-level timer expression** (the "cron") from that. The operator never
hand-writes two things that have to be kept in sync.

Why: the boundary list is already needed to compute the window. If the operator also
hand-wrote the timer, the two could drift apart — add a boundary, forget the timer, and you
get windows that don't line up with when the job actually runs. Generating one from the other
removes that whole class of mistake. (A raw-timer override exists for test environments only.)

One convenience: a single schedule entry can list several report types that share a timetable
(the three end-of-day reports, say). At startup the application quietly expands that into
separate independent schedules — one per report type — so at runtime there's no "group", just
simple one-report-type schedules.

---

## 6. What happens when a schedule fires

1. The scheduler wakes up (only **one** server instance does, even though many are running —
   the clustered timer guarantees this).
2. It works out the window for this firing, using the shape rules in §3, from the time it was
   *scheduled* for — never the wall-clock time it actually woke up. (For a normal firing these
   are the same; the distinction matters for crash recovery and for manual backfills, where a
   run must report the period it was scheduled for, not when it happened to execute.)
3. It hands the pipeline: **report type · frequency · scheduled time · window start · window
   end**, and creates one tracking record ("Run") for that firing.
4. The pipeline does everything else — find the matching report configurations, resolve their
   account/scope data (its own concern, described in `../data-retrieval/goal.md`), build the
   messages, publish them.

---

## 7. Design decisions worth knowing

**One worker, many timetables — not one worker per report type.** There's a single job type
that reads "which report, which frequency, which window" from its own configuration. Fourteen
timetables are registered from a loop, not fourteen hand-written classes.

**A small admin surface.** Authenticated, admin-only endpoints let an operator: run a schedule
now, backfill one or more missed slots (passing the slot times), pause a schedule, resume it,
and list schedule status. This is how a missed end-of-day report gets recovered (§8), and how
a schedule is stopped without a redeploy.

**Overlapping runs are allowed on purpose.** If the 12:30–13:00 run is slow and still going
when the 13:00–13:30 run fires, **both run at the same time and both publish**. A slow run
must never delay the next window's report. The cost — two runs reading the same data at once —
is accepted; the goal is to generate as fast as possible. (There's a monitoring signal if runs
*consistently* pile up, which means "make it faster or space it out".)

**A repeated firing for the *same* slot is a harmless no-op.** If the same scheduled slot
somehow fires twice (a timer hiccup), the second one notices a run already exists and exits
quietly — not an error.

**Recovery is split two ways, by how far a crash got.** If a server dies after a run has
already started being tracked, the pipeline's own recovery mechanism picks it up and continues
from where it stopped. If a server dies in the brief moment *before* that tracking even began,
the scheduler's own clustering re-fires that one firing — safely, because it either starts the
tracking fresh or, if tracking turns out to have already begun, simply recognizes that and
stops (the same harmless no-op as a repeated firing, above). The two never step on the same
work.

**Restart safety.** On redeploy, the timer store can hold stale timetables from the previous
configuration. At startup the application reconciles what's stored against the current config
and removes anything orphaned, *before* the scheduler starts running.

---

## 8. What happens when a firing is missed

A firing can go missing for more reasons than just an outage — every server being down is the
most common, but it also happens if an operator has deliberately paused that schedule, for
example to protect production while an environment issue is being worked. Whatever the cause,
it's handled the *same* way, and an operator always has the same option regardless of why a
slot was missed: trigger it explicitly. Pausing doesn't queue the firings it covers for later;
when the schedule is resumed, they're simply gone, exactly as if the system had been down for
that stretch.

**The automatic policy is: do nothing.** The missed slot is skipped entirely; the timetable
simply resumes at its next natural scheduled time and produces that window normally. There is
**no automatic catch-up**, for any report — frequent or end-of-day — and none released in a
burst when a paused schedule is resumed.

**Backfilling** is a first-class capability, available for **any** missed slot, sub-daily or
end-of-day alike — there's no restriction on which ones can be recovered. An operator (or a
small admin endpoint) can fire a job for one or more missed slots, passing the missed slot
times as parameters. Each backfill produces exactly the window that firing would have produced
on time, and is idempotent — a backfill for a slot that already ran is a harmless no-op.

Whether it's *worth* doing depends on the report:

- **Frequent, sub-daily reports** are often left alone: the next run is only minutes away and
  will cover the following window regardless, so the cost of one skipped window is low. But if
  that specific window matters, it can be backfilled exactly the same way as any other slot.
- **End-of-day reports** are different: nothing else will produce that day's statement, so a
  missed one is normally worth backfilling explicitly. Recommended: an alert — *"no end-of-day
  run for date D"* — so an operator knows to do that.

This is also the intended shape of a deliberate pause: pause the schedule to ride out the
environment issue, fix it, resume the schedule for firings going forward, then — once it's
agreed with the business which slots actually need recovering — backfill exactly those, by
their slot times. Nothing is auto-replayed in either case.

---

## 9. What's left before build

One check, not a design decision: a test that the scheduler skips a missed firing cleanly
under the "do nothing" policy — for both a true misfire (§8) and a trigger resumed after being
paused across one or more of its fire times (§8).
