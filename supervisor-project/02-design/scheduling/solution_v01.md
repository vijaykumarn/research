# Commander — Scheduling Design

How the **scheduled** trigger decides *when* each report runs and *what reporting window* a
firing represents, then hands off to the message-production pipeline
(`../message-pipeline/how-it-works.md` / `../message-pipeline/solution_v08.md`). Scheduling owns nothing after a `Run` is
created.

**In scope:** cadences, the reporting-window rules, wiring to Quartz, the deploy-time config
shape, and the admin endpoints (manual run, backfill, pause/resume, status).
**Out of scope:** producing the report messages (the pipeline); a schedule-management UI or
runtime editing of the *timetables themselves* (that stays deploy-time config); holiday
calendars.

> Visual: `scheduling-flow.drawio` (draw.io / diagrams.net) — page 1 the three trigger
> sources, page 2 the scheduled job lifecycle (with the misfire / backfill branch), page 3
> the three window shapes on a timeline, page 4 clustered Quartz across pods.

---

## 1. What a firing produces

Per firing, the job:

1. Takes the trigger's **scheduled** fire time — never wall-clock — or an explicit
   `scheduledTimeOverride` from its data map (see §8).
2. Resolves it to a `scheduled_time` and computes `(window_start, window_end)` via the window
   function (§4). `scheduled_time` is the *resolved boundary or fire instant*, not the raw
   Quartz fire time.
3. Starts **one pipeline `Run` for exactly one report type**:

> `report_type` · `frequency` · `scheduled_time` · `window_start` · `window_end`

The pipeline creates the `Run` — guarded by `UQ_Run_ScheduledSlot`, a filtered unique index on
`(report_type, frequency, scheduled_time)` over `SCHEDULED` rows (`../message-pipeline/solution_v08.md`) — and
pages `ReportConfig WHERE report_type = ? AND frequency = ? AND is_active = 1`, per
`../message-pipeline/how-it-works.md` §4. `frequency` and the window are persisted on the `Run` row: one report
type can have several frequencies (CAMT054C has three), and recovery's "keep paging" step
needs both to know which config set to page and what window to stamp.

A `Run` is single-report-type. A firing therefore always maps to **one `Run` for one report
type**, never several — where a config entry groups report types (§6), the loader has already
expanded that into separate per-report-type schedules before anything fires (§5).

---

## 2. Frequency catalogue

A `ReportConfig.frequency` value selects which schedule picks it up. All times are in the
single configured business timezone (`commander.scheduling.timezone = Europe/Stockholm`).
Business day = Mon–Fri, no holiday calendar.

| Report type(s) | `frequency` | Shape | Fires at (business TZ) | Days | Reporting window |
|---|---|---|---|---|---|
| CAMT052B, CAMT052BT | `EVERY_30_MIN` | rolling | every 30 min, 00:30 → 21:00 | Mon–Fri | `fire − 30 min → fire` |
| CAMT052B, CAMT052BT | `EVERY_1_HOUR` | rolling | hourly, 01:00 → 21:00 | Mon–Fri | `fire − 1 h → fire` |
| CAMT052B, CAMT052BT | `EVERY_2_HOURS` | boundary | 03:00, 05:00 … 21:00 | Mon–Fri | previous boundary → this; **first window 00:00 → 03:00** (3 h) |
| CAMT052B, CAMT052BT | `EVERY_4_HOURS` | boundary | 05:00, 09:00, 13:00, 17:00, 21:00 | Mon–Fri | previous boundary → this; **first window 00:00 → 05:00** (5 h) |
| CAMT054C | `ONCE_PER_DAY` | boundary | 21:00 | Mon–Fri | 00:00 → 21:00, same day |
| CAMT054C | `FOUR_TIMES_PER_DAY` | boundary | 10:00, 13:00, 18:00, 21:00 | Mon–Fri | previous boundary → this; first = 00:00 → 10:00 |
| CAMT054C | `EIGHT_TIMES_PER_DAY` | boundary | 03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00 | Mon–Fri | previous boundary → this; first = 00:00 → 03:00 |
| CAMT053S, CAMT053E, CAMT054D | `END_OF_DAY` | calendar-day | 06:00 | Tue–Sat | the whole **previous calendar day**, 00:00 → 24:00 |
| any of the above | `NEVER` | — | never scheduled | — | — |

- **`EVERY_30_MIN` / `EVERY_1_HOUR` are rolling.** Cheap, and rolling coincides with
  midnight-anchoring for these two — the first fire of the day is exactly one interval past
  midnight.
- **`EVERY_2_HOURS` / `EVERY_4_HOURS` are boundary** (decided — §12). The first window of the
  day starts at **00:00**, so midnight → first fire is covered. This is a **deliberate change
  from the legacy system**, which leaves 00:00 → 01:00 unreported for these two — a parity
  note for cutover.
- **`NEVER`** is the PHT-only marker (`../message-pipeline/solution_v08.md`) — the config exists so the PHT
  flow can resolve it; no scheduled trigger ever selects it. The scheduled selection predicate
  (`frequency = ?`) structurally excludes it.
- The 21:00 firing exists in all three CAMT054C cadences with a **different window each**; a
  config is on exactly one cadence, and the paging query filters on `frequency`, so they never
  collide.
- **`END_OF_DAY`** and **`ONCE_PER_DAY`** are deliberately separate values — same cadence,
  different window rule. See `../faq.md` Q6.
- **All boundary times are config values.** `EIGHT_TIMES_PER_DAY`'s set above is the starting
  value; any boundary list can be revised in config and takes effect on the next redeploy
  (startup reconciliation, §5, removes the superseded triggers). "Deploy-time, not runtime" —
  there is no live schedule editor (§6).

---

## 3. Handoff contract

Per firing, scheduling produces exactly:

`report_type` (one) · `frequency` · `scheduled_time` · `window_start` · `window_end`

The pipeline takes it from there. It derives the scheduled-path outbox `execution_id`
deterministically from these — `uuid5("SCHEDULED|{report_type}|{frequency}|{scheduled_time}")`
(`../message-pipeline/solution_v08.md`) — so the same slot (crash recovery, or a manual
backfill that passes that slot's time) dedups, while a genuinely distinct firing (a
fast-cadence TEST tick, an un-overridden manual re-run) produces a fresh message. This is what
lets §13's fast-cadence testing work for every frequency, `END_OF_DAY` included.

---

## 4. The window function

Pure, business-timezone, half-open `[start, end)`. Zone injected once. `Instant` output. Three
shapes.

### Rolling — `EVERY_30_MIN`, `EVERY_1_HOUR`

`end = scheduled fire time`, `start = end − interval`. No boundary list, no sequence.

### Boundary — `EVERY_2_HOURS`, `EVERY_4_HOURS`, `ONCE_PER_DAY`, `FOUR_TIMES_PER_DAY`, `EIGHT_TIMES_PER_DAY`

The frequency's ordered boundary list, with an implicit `00:00` prepended:

| `frequency` | Boundary list (implicit leading 00:00) |
|---|---|
| `EVERY_2_HOURS` | 00:00, 03:00, 05:00, 07:00 … 21:00 |
| `EVERY_4_HOURS` | 00:00, 05:00, 09:00, 13:00, 17:00, 21:00 |
| `FOUR_TIMES_PER_DAY` | 00:00, 10:00, 13:00, 18:00, 21:00 |
| `EIGHT_TIMES_PER_DAY` | 00:00, 03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00 |
| `ONCE_PER_DAY` | 00:00, 21:00 |

**Sequence resolution** (the single generated cron does not carry an index):

1. Take the scheduled fire time; convert to a local `(date, time)` in the business zone.
2. Find the largest real boundary `b` with `b ≤ time`.
   - Normal firing: `time` equals a boundary exactly. Under the do-nothing misfire policy (§9),
     this is the only way an *automatic* firing ever arrives.
   - Off-grid `time` (only reachable via a manual trigger with an odd `scheduledTimeOverride`):
     `b` is the most recent boundary before it, within a small tolerance (e.g. ±2 min). If
     `time` is not plausibly attributable to a boundary, or `b` resolves to the implicit
     leading `00:00` → **log and skip** (no `Run` created).
3. `window = (previous boundary, b]`, resolved against `date`. The first real boundary of the
   day → `window_start = 00:00`.
4. `scheduled_time = b` on `date`.

*Examples:* `FOUR_TIMES_PER_DAY` at 10:00 → `00:00–10:00`; at 13:00 → `10:00–13:00`; at 21:00
→ `18:00–21:00`. `ONCE_PER_DAY` at 21:00 → `00:00–21:00`.

### Calendar-day — `END_OF_DAY`

`scheduled_time` = the 06:00 fire instant. `window` = `00:00` to `24:00` (business zone) of
the **calendar day before** `scheduled_time`'s local date. A backfill for a missed
`END_OF_DAY` (§8) passes that date's 06:00 as `scheduledTimeOverride`, so it covers exactly
the calendar day it would have on time.

### Daylight saving

The business timezone observes DST: clocks jump **forward** one hour in spring (local time
skips 02:00 → 03:00) and **back** one hour in autumn (local time repeats 01:00 → 02:00). Three
consequences for boundary scheduling, with the chosen rules:

- **A boundary in the spring-forward gap** (e.g. a future boundary at 02:00 or 02:30, which
  simply doesn't occur on that day) → **shift it forward to the first instant that does
  exist** (03:00), and fire there. A plain cron would just not fire for a non-existent time,
  silently dropping that boundary's run and leaving the next window starting from a boundary
  that never happened. Shifting keeps every window covered; the cost is a one-day, one-hour
  distortion of that boundary once a year.
- **A boundary in the fall-back overlap** (occurs twice) → use the **first (earlier)**
  occurrence and fire **once**. Deterministic, and it prevents a doubled run / double-counted
  window.
- **A window that merely spans a transition** (no boundary in the gap) is a *calendar* window,
  not a fixed duration — `EVERY_2_HOURS`' `00:00–03:00` is ~2 elapsed hours on spring-forward
  day and ~4 on fall-back day; `END_OF_DAY` spans **23 h or 25 h**. This is intended. The
  message contract should note a window's elapsed length can differ by an hour on the two
  transition days so Executor isn't surprised.

**Current boundary set is safe** — no boundary falls in `02:00–03:00`, so none of the above
triggers today. But boundary lists are editable config (§2), so the rules must be in the
window function before anyone adds an early-hours boundary.

Implementation: Java's `ZonedDateTime.of(date, localTime, zone)` already resolves
gap → shift-forward and overlap → earlier-offset, which matches the rules above — but pin it
with a **named resolver and tests**, so it's a decision, not a library default that could
change.

---

## 5. Trigger model

**One logical schedule per `(report_type, frequency)`.** Rough counts: ~8 config entries →
**14 logical schedules** → **~16 physical `CronTrigger`s** (14 + one extra `EVERY_30_MIN` cron
× 2 report types — see below).

**One job class.** `ReportSchedulingJob implements Job`. The differentiators —
`report_type`, `frequency`, the window spec — live in each JobDetail's own `JobDataMap`, not
in a Java subclass. The loader builds **one `JobDetail` per `(report_type, frequency)`** in a
loop (14 total), each in its report type's group (e.g. `camt052b-group`).

- `storeDurably(true)`.
- `requestRecovery(false)` — **the pipeline's own recovery owns interrupted runs** (heartbeat
  + stale-`Run` sweeper, `../message-pipeline/solution_v08.md`). Quartz's `requestRecovery` re-fires the
  *trigger* (a fresh firing); the sweeper resumes the *specific `Run`* from its checkpoint.
  Two overlapping mechanisms would race — use only the sweeper.

**Grouping is config sugar.** A config entry may list several `report-types`; the loader
**expands** it into one independent registration per report type at startup. Nothing fans out
at runtime.

**Crons are generated, never hand-authored alongside a boundary list.** Each schedule is
authored once as a declarative spec — an interval (`first` / `step` / `last`) or an explicit
boundary list, plus `days` — and the loader derives the cron expression(s). No two sources of
truth to drift.

Generated crons (business TZ, `.inTimeZone(zone)` always set explicitly — Quartz defaults to
the JVM zone otherwise):

| `frequency` | Generated cron(s) |
|---|---|
| `EVERY_1_HOUR` | `0 0 1-21 ? * MON-FRI` |
| `EVERY_2_HOURS` | `0 0 3-21/2 ? * MON-FRI` |
| `EVERY_4_HOURS` | `0 0 5-21/4 ? * MON-FRI` |
| `EVERY_30_MIN` | `0 30 0-20 ? * MON-FRI` + `0 0 1-21 ? * MON-FRI` (two crons — no single cron hits exactly 00:30 … 21:00) |
| `ONCE_PER_DAY` | `0 0 21 ? * MON-FRI` |
| `FOUR_TIMES_PER_DAY` | `0 0 10,13,18,21 ? * MON-FRI` |
| `EIGHT_TIMES_PER_DAY` | `0 0 3,6,8,10,12,15,18,21 ? * MON-FRI` |
| `END_OF_DAY` | `0 0 6 ? * TUE-SAT` |

**Boundary frequencies use a single cron**, not one trigger per boundary. A single cron works
whenever a frequency's boundary times share a minute (all `:00` in the current catalogue). If
a future boundary sits on a different minute, the loader emits **one cron per distinct minute
value** — still far fewer than one per boundary.

**Orphaned-trigger reconciliation.** Quartz's persistent store retains triggers from a prior
config after a redeploy. A startup runner (with `spring.quartz.auto-startup=false`) diffs the
persisted triggers in each report-type group against the current config, removes orphans, then
starts the scheduler — so nothing fires against a stale trigger. Only the per-report-type
groups are inspected, so unrelated Quartz functionality is untouched. This is
cluster-tolerant: `unscheduleJob` returning false (another node removed it first) is expected.

---

## 6. Configuration (deploy-time, via properties)

Fixed at deploy time, changed by redeploy. Each entry carries only the meaningful inputs; the
loader derives the crons. Illustrative:

```properties
commander.scheduling.timezone = Europe/Stockholm

# interval-spec form
commander.scheduling.triggers[0].report-types  = CAMT052B, CAMT052BT
commander.scheduling.triggers[0].frequency      = EVERY_2_HOURS
commander.scheduling.triggers[0].days           = MON-FRI
commander.scheduling.triggers[0].shape          = BOUNDARY         # ROLLING | BOUNDARY | CALENDAR_DAY
commander.scheduling.triggers[0].interval       = { first: 03:00, step: 2h, last: 21:00 }
                                                 # implicit leading 00:00 → first window is 00:00–03:00

# explicit boundary-list form
commander.scheduling.triggers[4].report-types   = CAMT054C
commander.scheduling.triggers[4].frequency      = EIGHT_TIMES_PER_DAY
commander.scheduling.triggers[4].days           = MON-FRI
commander.scheduling.triggers[4].shape          = BOUNDARY
commander.scheduling.triggers[4].boundaries     = 03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00

# calendar-day
commander.scheduling.triggers[7].report-types   = CAMT053S, CAMT053E, CAMT054D
commander.scheduling.triggers[7].frequency      = END_OF_DAY
commander.scheduling.triggers[7].days           = TUE-SAT
commander.scheduling.triggers[7].shape          = CALENDAR_DAY
commander.scheduling.triggers[7].fire-at        = 06:00

# optional — TEST only: override the generated wake-up schedule with a raw cron
commander.scheduling.triggers[0].cron-override  = 0 0/2 * ? * MON-FRI
```

- **No hand-written `cron` field in normal use**, no day-of-week baked into a cron string.
  `days` is the single source of truth for day-of-week; `interval` / `boundaries` for the
  times; `fire-at` for `CALENDAR_DAY`.
- `interval` and `boundaries` are interchangeable for any `BOUNDARY` shape — use whichever is
  less to type.
- **`cron-override`** (optional, TEST): when present the loader registers it verbatim (still
  `.inTimeZone(zone)`) instead of generating. The boundary list still governs the window via
  §4's sequence resolution; the off-grid "skip" guard is disabled under an override. Absent in
  production.

---

## 7. Concurrent runs are intended

Two firings for the same `(report_type, frequency)` — an in-flight slow run for 12:30–13:00
and the next firing for 13:00–13:30 — **run concurrently, and both should publish.** A slow
run must never hold back the next window's report.

The pipeline handles this correctly with no locking:

- The two `Run`s have **different `scheduled_time`** (13:00 vs 13:30), so
  `UQ_Run_ScheduledSlot` does not block them — it only blocks a *duplicate* firing for the
  *same* slot (a misfire double-fire).
- Each run makes its own `WorkItem` rows (keyed by `run_id`) and its messages carry
  **different `window_start` / `window_end`** → different `UQ_Outbox_Identity` → both publish.

**A duplicate firing for the same slot is a named success path, not a failure.** If a firing
finds a `Run` already exists for its `(report_type, frequency, scheduled_time)`, the job
catches the `UQ_Run_ScheduledSlot` violation, logs it, and **exits cleanly** — it does not
throw to Quartz (a `JobExecutionException` would be recorded as a failed execution and could
escalate to misfire/alert handling). The existing `Run` is either `COMPLETED` (idempotent
no-op), `IN_PROGRESS` (the recovery sweeper owns it), or `ABANDONED` (the give-up alert
already fired — re-running that slot is a manual action, §8). Same posture as
`../message-pipeline/solution_v08.md`'s `Outbox` "already exists" success path.

**No `@DisallowConcurrentExecution` on `ReportSchedulingJob`.** The double DB read load from
two overlapping runs paging the same configs is accepted — the goal is a service that
generates as fast as it can.

**Operational signal, not a code block:** if a frequency's runs *routinely* overlap (every
`EVERY_30_MIN` run consistently taking > 30 min), they pile up — N concurrent runs all paging
the same configs is a thundering herd. Alert on **≥ K concurrent `IN_PROGRESS` runs for one
`(report_type, frequency)`** as a "make it faster or widen the interval" signal.

---

## 8. Manual execution

A manual run is `scheduler.triggerJob(JobKey)`, keyed by one of the 14 JobDetails:

```java
scheduler.triggerJob(JobKey.jobKey("CAMT052B-EVERY_30_MIN", "camt052b-group"));
```

Runs that job now, out of band from its cron. Because the JobDetail's own data map carries
`report_type` + `frequency` + the window spec, the job has everything it needs — no trigger
context required.

**Re-run a specific slot** — `ReportSchedulingJob` reads an optional override from its merged
data map:

```java
Instant scheduledTime = data.containsKey("scheduledTimeOverride")
    ? Instant.parse(data.getString("scheduledTimeOverride"))
    : context.getScheduledFireTime().toInstant();
```

```java
JobDataMap override = new JobDataMap();
override.put("scheduledTimeOverride", "2026-09-09T13:00:00Z");
scheduler.triggerJob(JobKey.jobKey("CAMT052B-EVERY_1_HOUR", "camt052b-group"), override);
```

Re-runs the 13:00 slot (window 12:00–13:00) whenever fired. Without the override, a manual
fire uses "now" resolved through §4.

**Day-to-day surface** — the `POST /admin/scheduling/run` endpoint (full list in §14) wraps
`triggerJob`:

```
POST /admin/scheduling/run
{ "reportType": "CAMT053S", "frequency": "END_OF_DAY", "scheduledTime": "2026-09-08T06:00:00Z" }
```

→ resolves the JobKey, calls `triggerJob(key, overrideMap)`.

### Backfilling missed slots

Because the misfire policy is **do-nothing** (§9), a firing missed while the cluster was down
is *never* recovered automatically — it is backfilled by an explicit manual trigger. Each
backfill is just a manual run (above) with `scheduledTimeOverride` set to the missed slot's
time, so the window function resolves it exactly as if it had fired on time.

For **several** missed slots, `POST /admin/scheduling/backfill` (§14) takes either an explicit
list of slot times or a `(from, to)` range it expands into that frequency's boundaries:

```
POST /admin/scheduling/backfill
{ "reportType": "CAMT054C", "frequency": "FOUR_TIMES_PER_DAY",
  "from": "2026-09-10T00:00:00Z", "to": "2026-09-10T18:30:00Z" }
```

→ expands to the boundaries in `[from, to]` (10:00, 13:00, 18:00), fires **one job per
boundary**, each with its own `scheduledTimeOverride`. Every backfill produces an independent
`Run` for its slot and is fully idempotent — `UQ_Run_ScheduledSlot` makes a re-issued backfill
for a slot that already ran a clean no-op. `END_OF_DAY` backfill is a single slot: the missed
date's 06:00.

**Boundary with the on-demand path:** *"re-run scheduled slot X"* / *"backfill missed slots"*
is the manual-trigger path above (still goes through the scheduled pipeline — window computed
from the slot, `Run` created and deduped, publish). *"Generate for an arbitrary historical
window or a specific config-id list"* is the **on-demand trigger's** job — don't force an
arbitrary window into the scheduled job.

---

## 9. Misfire — decided: do nothing

**`MISFIRE_INSTRUCTION_DO_NOTHING` on every scheduled trigger, all frequencies.** If the
whole cluster was down across a firing, that slot is **skipped entirely** — no automatic
catch-up, ever. The trigger simply resumes at its next natural scheduled time and produces
that window normally.

Consequences, all accepted:

- Sub-daily reports lose one window (30 min to a few hours). The next firing is close behind.
- **`END_OF_DAY` / `ONCE_PER_DAY` lose a whole day's report** if the cluster is down across
  06:00 / 21:00. Recommended (not required): an alert — *"no `END_OF_DAY` run recorded for
  date D"* — so an operator knows to backfill it manually (§8).
- Any missed slot — including a full day's `END_OF_DAY` — is recovered **only** by an explicit
  manual / batch backfill (§8, *Backfilling missed slots*). Backfills carry the missed slot
  time as `scheduledTimeOverride`, so the window is identical to what the on-time firing would
  have produced, and `UQ_Run_ScheduledSlot` keeps them idempotent.

Design payoff: because only on-grid firings ever happen, §4's sequence resolution is always an
exact boundary match; the off-grid snap/skip guard is purely defensive (it only fires if a
manual trigger passes an odd `scheduledTimeOverride`).

**Verify with a real Quartz test** under a clustered `JDBCJobStore` that
`MISFIRE_INSTRUCTION_DO_NOTHING` on a `CronTrigger` skips the missed firing cleanly and the
next scheduled fire is unaffected.

---

## 10. Interaction with pipeline recovery

- Scheduling creates the `Run` as the job's **first step**, before any config resolution — so
  a firing that fails partway still leaves a trace the recovery sweeper can act on. A firing
  that fails *before* `Run` creation (e.g. a bad config) is caught by startup validation
  (§11), not left silent.
- The `Run` row records `frequency` and the resolved `(window_start, window_end)` — the
  sweeper resuming a run past `last_config_id_processed` reads `frequency` to know which config
  set to keep paging and the stored window to stamp onto new `WorkItem`s (no recompute).
- The recovery sweeper (`../message-pipeline/solution_v08.md`) owns interrupted scheduled runs — heartbeat
  detection, CAS ownership, resume-from-checkpoint. Scheduling adds nothing here beyond
  creating the `Run` and letting `requestRecovery(false)` keep Quartz out of it.
- `UQ_Run_ScheduledSlot` makes a repeated firing for the same slot fail fast at `Run` creation
  instead of after a full resolve pass; the job treats that violation as a clean no-op (§7).

---

## 11. Startup validation

At startup, fail loud (or alert) on:

- A schedule entry whose `frequency` string does not parse to a known value.
- A schedule entry with `shape = BOUNDARY` whose boundary list is empty, not strictly
  ascending, or includes `00:00`.
- A schedule entry with neither an interval spec nor a boundary list (or, for
  `shape = CALENDAR_DAY`, no `fire-at`).
- A duplicate `(report_type, frequency)` across entries.
- Any `(report_type, frequency)` present in `ReportConfig` (excluding `NEVER`) with **no
  matching schedule** — those configs would silently never be produced.

---

## 12. `EVERY_2_HOURS` / `EVERY_4_HOURS` window shape — decided: boundary

**Decision: `shape = BOUNDARY`, midnight-anchored.** The first window of the day is
`00:00 → 03:00` (2 h freq) and `00:00 → 05:00` (4 h freq); every later firing is
previous-boundary → this-boundary as normal.

**Parity note for cutover.** The legacy system runs these two as a rolling look-back, so its
first window is `01:00 → 03:00` / `01:00 → 05:00` and **00:00 → 01:00 is not reported** each
day. Commander deliberately closes that gap. This also brings `EVERY_2/4_HOURS` in line with
CAMT054C's boundary frequencies, which already cover from midnight — removing an
inconsistency the legacy system carries between the two families.

Fire times are unchanged (03:00, 05:00 … 21:00 and 05:00, 09:00, 13:00, 17:00, 21:00), so the
generated crons in §5 are unaffected; only the window computation moves from rolling to
boundary.

---

## 13. Testing on a fast cadence

Give the TEST profile a **dense spec** — `interval = { first: 00:00, step: 2m, last: 23:58 }`
or a dense `boundaries` list, or a `cron-override` — and every fire produces a fresh message.

This works for **every frequency, including `END_OF_DAY`**, because a scheduled outbox row's
`execution_id` is derived from its `scheduled_time`
(`uuid5("SCHEDULED|{report_type}|{frequency}|{scheduled_time}")`, `../message-pipeline/solution_v08.md`). Each
tick of a fast cron has a distinct `scheduled_time` → a distinct `execution_id` → a new
`UQ_Outbox_Identity`, so a fresh message is published even when the *window* is identical.

- For rolling / boundary frequencies the window also advances, so this is doubly fresh.
- For `END_OF_DAY` the window stays "yesterday 00:00–24:00" on every tick, but the message is
  still new each time — you can fire it every 15 minutes in TEST and get a full round-trip to
  Executor on each fire.

Production is unaffected: `END_OF_DAY` fires once a day, and crash recovery / a manual backfill
reuse the slot's `scheduled_time` (so the same `execution_id`) and dedup normally.

---

## 14. Admin endpoints

One authenticated, admin-only surface (a custom Actuator endpoint or a small guarded
controller). All operations key on `(report_type, frequency)` — the identity of a logical
schedule.

| Endpoint | Purpose | Body / params | Behaviour |
|---|---|---|---|
| `POST /admin/scheduling/run` | Fire one schedule now, out of band | `reportType`, `frequency`, optional `scheduledTime` | `triggerJob(key, override)`. With `scheduledTime` → that slot's window; without → "now" resolved through §4. |
| `POST /admin/scheduling/backfill` | Recover one or more missed slots | `reportType`, `frequency`, and either `slots: [...]` or `from` + `to` | Expands `(from, to)` into that frequency's boundaries in range (or takes the explicit list); fires **one job per slot**, each with its own `scheduledTimeOverride`. Idempotent via `UQ_Run_ScheduledSlot`. |
| `POST /admin/scheduling/pause` | Pause a schedule | `reportType`, `frequency` | `scheduler.pauseTrigger(key)` for **every** `TriggerKey` of that schedule (`EVERY_30_MIN` has two). Persisted in the JobStore → cluster-wide, survives restarts. |
| `POST /admin/scheduling/resume` | Resume a paused schedule | `reportType`, `frequency` | `scheduler.resumeTrigger(key)` for all its `TriggerKey`s. **Forward-only** — firings missed while paused are *not* backfilled (they follow the §9 do-nothing rule); use `backfill` if you need them. |
| `GET /admin/scheduling/status` | Inspect schedules | — | Lists each `(report_type, frequency)`: paused/active, its `TriggerKey`s, last fire time, next fire time. |

Pause/resume is **native Quartz, no new state** — `pause`/`resume` just toggle the clustered
scheduler's trigger state. A paused trigger stays paused across a redeploy as long as it's
still in config; startup reconciliation (§5) removes it only if it's been dropped from config.

---

## 15. Edge cases

- **Business day = Mon–Fri, no holiday calendar.** A weekday holiday still runs its firings
  and reports that (near-empty) day; normal empty-report handling applies.
- **Every `CronTrigger` sets `.inTimeZone(businessZone)` explicitly.**
- **21:00 → 24:00 is never reported** by any boundary or rolling frequency.
- **`NEVER` configs** never appear in a scheduled `Run`.
- **Physical triggers for one logical schedule have disjoint fire times** — they never
  double-fire the same boundary.
- **A firing whose scheduled time can't be attributed to a boundary** (implausible off-grid,
  or resolves to the implicit `00:00`) is logged and skipped, not forced onto a wrong window.
- **A firing that matches no active configs** (`report_type`/`frequency` combination has no
  `is_active = 1` rows) completes as a zero-`WorkItem` `Run` — a clean no-op, not an error.
- **Changing a boundary list** is a config edit + redeploy; startup reconciliation (§5) drops
  the superseded triggers before the scheduler starts.

---

## 16. Status of decisions

**Decided:**

- Misfire policy — **do nothing** (§9). Missed slots recovered only by explicit
  manual/batch backfill (§8, §14).
- `EVERY_2_HOURS` / `EVERY_4_HOURS` — **boundary**, first window anchored to `00:00` (§12) — a
  deliberate divergence from the legacy system's rolling behaviour (parity note for cutover).
- `EIGHT_TIMES_PER_DAY` boundaries — `03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00`
  (§2), a config value that can be revised in a later release (edit + redeploy).
- Crons are **generated** from the declarative spec, not hand-written (§5, §6). A raw
  `cron-override` exists for TEST only.
- **Pause / resume ships in v1**, alongside manual-run and backfill, as admin endpoints (§14).
- DST — spring-forward boundary → shift forward; fall-back boundary → first occurrence;
  transition-day windows are calendar windows, not fixed durations (§4).

- Business timezone — `Europe/Stockholm`.

**Still to confirm / verify (implementation, not architecture):**

- The 8 `EIGHT_TIMES_PER_DAY` times and the DST rule — final sign-off from the business.
- A Quartz test that `MISFIRE_INSTRUCTION_DO_NOTHING` skips a missed `CronTrigger` firing
  cleanly and the next fire is unaffected.
