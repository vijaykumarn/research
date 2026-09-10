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
`EIGHT_TIMES_PER_DAY`, `ONCE_PER_DAY`) **and, now, the every-2-hours / every-4-hours intraday
reports** — so their first run of the day covers from midnight (00:00–03:00 / 00:00–05:00),
not from an hour in.

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

Two housekeeping points:

- **Timezone.** Every time above is in one configured business timezone. All the window maths
  happens in that zone, then converts to absolute time for the message.
- **The "never" marker.** Some report configurations exist only for the external balance-push
  (PHT) flow and must never be picked up by the scheduler. They carry a special frequency
  value (`NEVER`) that no schedule matches, so they're excluded automatically — no special
  code.

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
   data, build the messages, publish them.

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

**Recovery is the pipeline's job, not the scheduler's.** If a server dies mid-run, the
pipeline's own recovery mechanism picks it up and continues from where it stopped. The
scheduler deliberately does *not* also try to re-fire — two recovery mechanisms would fight.

**Restart safety.** On redeploy, the timer store can hold stale timetables from the previous
configuration. At startup the application reconciles what's stored against the current config
and removes anything orphaned, *before* the scheduler starts running.

---

## 8. What happens if the whole system was down

If every server was down across a scheduled firing, that firing is *missed*. **The policy is:
do nothing.** The missed slot is skipped entirely; the timetable simply resumes at its next
natural scheduled time and produces that window normally. There is **no automatic catch-up**,
for any report — frequent or end-of-day.

- **Frequent reports (sub-daily):** losing one window is cheap; the next run is minutes away.
- **End-of-day reports:** a missed 06:00 firing means that day's statement is not produced. It
  must be **backfilled by an explicit manual trigger**. Recommended: an alert — *"no
  end-of-day run for date D"* — so an operator knows to do that.

**Backfilling** is a first-class capability: an operator (or a small admin endpoint) can fire
a job for one or more missed slots, passing the missed slot times as parameters. Each backfill
produces exactly the window that firing would have produced on time, and is idempotent — a
backfill for a slot that already ran is a harmless no-op.

Deliberately *not* doing: any automatic replay of missed slots. Recovery of a gap is always a
conscious operator action, never a surprise burst of catch-up runs after an outage.

---

## 9. Open points for the architect

Almost everything is now settled. Two items await a final business sign-off (not a redesign):

| # | Item | Status |
|---|---|---|
| 1 | **Daylight-saving rules** — a boundary time that doesn't exist (spring) shifts forward to the next real instant; one that happens twice (autumn) uses the earlier occurrence; transition-day windows are calendar windows, so their elapsed length can be an hour short/long twice a year. | Rules chosen; confirm they match business expectation. |
| 2 | **The eight fire times** for the 8×-a-day notification report — starting value `03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00`, revisable later via config. | Confirm the starting set. |

Plus one build-time check: a test that the scheduler skips a missed firing cleanly under the
"do nothing" policy.

**Decided since the last review:**

- **Missed-firing policy — do nothing.** No automatic catch-up; missed slots are recovered by
  an explicit backfill (§8) — a first-class admin capability, not a workaround.
- **Every-2-hours / every-4-hours window — boundary, anchored to midnight.** First run of the
  day covers 00:00–03:00 / 00:00–05:00. This deliberately differs from the legacy system,
  which leaves 00:00–01:00 unreported for these two — a **parity note for cutover**.
- **Cron is generated** from the plain-terms config, never hand-written (a raw override exists
  for test only).
- **Pause / resume is in v1**, alongside run and backfill, as admin endpoints.

Everything else — the three shapes, the single-worker model, concurrent runs, the
scheduler/pipeline split, restart reconciliation — is settled.
