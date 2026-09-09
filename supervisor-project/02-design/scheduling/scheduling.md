# Commander — Scheduling Design

How the **scheduled** trigger decides *when* each report runs and *what reporting window* a
firing represents, then hands off to the message-production pipeline
(`../how-it-works.md` / `../solutions_v08.md`). Scheduling owns nothing after a `Run` is
created.

**In scope:** cadences, the reporting-window rules, wiring to Quartz, the deploy-time config
shape, manual execution, misfire policy, an optional pause/resume hook.
**Out of scope:** producing the report messages (the pipeline); any schedule-management UI or
runtime schedule editing; holiday calendars.

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

The pipeline creates the `Run` — guarded by `UNIQUE (report_type, frequency, scheduled_time)` —
and pages `ReportConfig WHERE report_type = ? AND frequency = ? AND is_active = 1`, per
`../how-it-works.md` §4.

A `Run` is single-report-type. A firing therefore always maps to **one `Run` for one report
type**, never several — where a config entry groups report types (§6), the loader has already
expanded that into separate per-report-type schedules before anything fires (§5).

---

## 2. Frequency catalogue

A `ReportConfig.frequency` value selects which schedule picks it up. All times are in the
single configured business timezone. Business day = Mon–Fri, no holiday calendar.

| Report type(s) | `frequency` | Shape | Fires at (business TZ) | Days | Reporting window |
|---|---|---|---|---|---|
| CAMT052B, CAMT052BT | `EVERY_30_MIN` | rolling | every 30 min, 00:30 → 21:00 | Mon–Fri | `fire − 30 min → fire` |
| CAMT052B, CAMT052BT | `EVERY_1_HOUR` | rolling | hourly, 01:00 → 21:00 | Mon–Fri | `fire − 1 h → fire` |
| CAMT052B, CAMT052BT | `EVERY_2_HOURS` | rolling *or* boundary — see §12 | 03:00, 05:00 … 21:00 | Mon–Fri | rolling `fire − 2 h → fire`, **or** boundary with first window `00:00 → 03:00` |
| CAMT052B, CAMT052BT | `EVERY_4_HOURS` | rolling *or* boundary — see §12 | 05:00, 09:00, 13:00, 17:00, 21:00 | Mon–Fri | rolling `fire − 4 h → fire`, **or** boundary with first window `00:00 → 05:00` |
| CAMT054C | `ONCE_PER_DAY` | boundary | 21:00 | Mon–Fri | 00:00 → 21:00, same day |
| CAMT054C | `FOUR_TIMES_PER_DAY` | boundary | 10:00, 13:00, 18:00, 21:00 | Mon–Fri | previous boundary → this; first = 00:00 → 10:00 |
| CAMT054C | `EIGHT_TIMES_PER_DAY` | boundary | 8 configurable times, last = 21:00 | Mon–Fri | previous boundary → this; first = 00:00 → first time |
| CAMT053S, CAMT053E, CAMT054D | `END_OF_DAY` | calendar-day | 06:00 | Tue–Sat | the whole **previous calendar day**, 00:00 → 24:00 |
| any of the above | `NEVER` | — | never scheduled | — | — |

- **`EVERY_30_MIN` / `EVERY_1_HOUR` are rolling.** Cheap, and rolling coincides with
  midnight-anchoring for these two — the first fire of the day is exactly one interval past
  midnight.
- **`EVERY_2_HOURS` / `EVERY_4_HOURS`** — shape is an **open decision** (§12). Rolling matches
  current production (00:00 → first fire is not covered); boundary anchors the first window to
  midnight.
- **`NEVER`** is the PHT-only marker (`../solutions_v08.md`) — the config exists so the PHT
  flow can resolve it; no scheduled trigger ever selects it. The scheduled selection predicate
  (`frequency = ?`) structurally excludes it.
- The 21:00 firing exists in all three CAMT054C cadences with a **different window each**; a
  config is on exactly one cadence, and the paging query filters on `frequency`, so they never
  collide.
- **`END_OF_DAY`** and **`ONCE_PER_DAY`** are deliberately separate values — same cadence,
  different window rule. See `../faq.md` Q6.

---

## 3. Handoff contract

Per firing, scheduling produces exactly:

`report_type` (one) · `frequency` · `scheduled_time` · `window_start` · `window_end`

The pipeline takes it from there.

---

## 4. The window function

Pure, business-timezone, half-open `[start, end)`. Zone injected once. `Instant` output. Three
shapes.

### Rolling — `EVERY_30_MIN`, `EVERY_1_HOUR`

`end = scheduled fire time`, `start = end − interval`. No boundary list, no sequence.

### Boundary — `ONCE_PER_DAY`, `FOUR_TIMES_PER_DAY`, `EIGHT_TIMES_PER_DAY` (and `EVERY_2/4_HOURS` if §12 lands on boundary)

The frequency's ordered boundary list, with an implicit `00:00` prepended:

| `frequency` | Boundary list (implicit leading 00:00) |
|---|---|
| `EVERY_2_HOURS` | 00:00, 03:00, 05:00, 07:00 … 21:00 |
| `EVERY_4_HOURS` | 00:00, 05:00, 09:00, 13:00, 17:00, 21:00 |
| `FOUR_TIMES_PER_DAY` | 00:00, 10:00, 13:00, 18:00, 21:00 |
| `EIGHT_TIMES_PER_DAY` | 00:00, *t₁ … t₇*, 21:00 |
| `ONCE_PER_DAY` | 00:00, 21:00 |

**Sequence resolution** (the single generated cron does not carry an index):

1. Take the scheduled fire time; convert to a local `(date, time)` in the business zone.
2. Find the largest real boundary `b` with `b ≤ time`.
   - Normal firing: `time` equals a boundary exactly.
   - Off-grid firing (a misfire whose scheduled time isn't a boundary): `b` is the most recent
     boundary before it, within a small tolerance (e.g. ±2 min). If `time` is not plausibly
     attributable to a boundary, or `b` resolves to the implicit leading `00:00` → **log and
     skip** (no `Run` created).
3. `window = (previous boundary, b]`, resolved against `date`. The first real boundary of the
   day → `window_start = 00:00`.
4. `scheduled_time = b` on `date`.

*Examples:* `FOUR_TIMES_PER_DAY` at 10:00 → `00:00–10:00`; at 13:00 → `10:00–13:00`; at 21:00
→ `18:00–21:00`. `ONCE_PER_DAY` at 21:00 → `00:00–21:00`.

### Calendar-day — `END_OF_DAY`

`scheduled_time` = the 06:00 fire instant. `window` = `00:00` to `24:00` (business zone) of
the **calendar day before** `scheduled_time`'s local date. A late (misfired) `END_OF_DAY`
firing uses its *scheduled* 06:00 date, so the covered day is unchanged.

### Daylight saving

Boundary times are wall-clock local times, resolved against the firing's local date:

- **Spring-forward gap** (a boundary time that doesn't exist that day) → resolve to the next
  instant that does exist. A skipped boundary would leave a permanently unreported slice of
  the day.
- **Fall-back overlap** (a boundary time that occurs twice) → resolve to the **first**
  occurrence.
- `END_OF_DAY`'s previous-calendar-day window spans **23 h or 25 h** on a transition day —
  intended (a calendar day, not a fixed duration).

Named, tested rules in the window function — not whatever the time library defaults to.

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
  + stale-`Run` sweeper, `../solutions_v08.md`). Quartz's `requestRecovery` re-fires the
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
| `EIGHT_TIMES_PER_DAY` | `0 0 <t1>,…,21 ? * MON-FRI` |
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
commander.scheduling.timezone = <business zone>

# interval-spec form
commander.scheduling.triggers[0].report-types  = CAMT052B, CAMT052BT
commander.scheduling.triggers[0].frequency      = EVERY_2_HOURS
commander.scheduling.triggers[0].days           = MON-FRI
commander.scheduling.triggers[0].shape          = ROLLING          # ROLLING | BOUNDARY | CALENDAR_DAY
commander.scheduling.triggers[0].interval       = { first: 03:00, step: 2h, last: 21:00 }

# explicit boundary-list form
commander.scheduling.triggers[4].report-types   = CAMT054C
commander.scheduling.triggers[4].frequency      = EIGHT_TIMES_PER_DAY
commander.scheduling.triggers[4].days           = MON-FRI
commander.scheduling.triggers[4].shape          = BOUNDARY
commander.scheduling.triggers[4].boundaries     = <t1>, <t2>, ... , 21:00

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
  `UNIQUE (report_type, frequency, scheduled_time)` does not block them — it only blocks a
  *duplicate* firing for the *same* slot (a misfire double-fire).
- Each run makes its own `WorkItem` rows (keyed by `run_id`) and its messages carry
  **different `window_start` / `window_end`** → different `UQ_Outbox_Identity` → both publish.

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

**Day-to-day surface** — a thin admin endpoint (custom Actuator endpoint or a small
authenticated `POST`) wrapping `triggerJob`:

```
POST /admin/scheduling/run
{ "reportType": "CAMT053S", "frequency": "END_OF_DAY", "scheduledTime": "2026-09-08T06:00:00Z" }
```

→ resolves the JobKey, calls `triggerJob(key, overrideMap)`.

**Boundary with the on-demand path:** *"re-run scheduled slot X"* is the manual-trigger path
above (still goes through the scheduled pipeline — window computed, `Run` created and deduped,
publish). *"Generate for an arbitrary historical window or a specific config-id list"* is the
**on-demand trigger's** job — don't force an arbitrary window into the scheduled job.

---

## 9. Misfire — OPEN decision

The policy value is a business decision. Mechanism and recommendation recorded.

**Recommendation:** `MISFIRE_INSTRUCTION_DO_NOTHING` on the sub-daily frequencies — if the
cluster was down across a firing, skip that slot and resume at the next boundary. Losing 30
minutes (or up to a few hours) of one window is cheap; the next firing produces the next
window normally. This keeps the design simple: only on-grid firings ever happen, so §4's
sequence resolution is exact and the off-grid guard is purely defensive.

**`END_OF_DAY` / `ONCE_PER_DAY` need a separate answer.** `DO_NOTHING` there means a whole
day's report is never produced because the cluster blipped at 06:00. Options:
`MISFIRE_INSTRUCTION_FIRE_ONCE_NOW` (fire once late — the window is still correct because
`END_OF_DAY` uses the *scheduled* date, not the fire time), or `DO_NOTHING` plus a separate
"was today's `END_OF_DAY` run produced?" check that alerts, with a manual re-run (§8) or an
on-demand backfill as the remedy.

If any catch-up *is* wanted for the boundary frequencies: catch up the **most recent missed
boundary only** (one `Run`, `(previous boundary, most-recent-missed boundary]`) — do not
replay every missed slot; older ones are a manual/on-demand backfill. Idempotency is free: the
catch-up's resolved `scheduled_time` and window equal what the on-time fire would have
produced, so `UNIQUE (report_type, frequency, scheduled_time)` and `UQ_Outbox_Identity`
reconcile it against any partial on-time run.

**Verify with a real Quartz test** under a clustered `JDBCJobStore` — confirm what
`getScheduledFireTime()` returns for each misfire instruction, and that §4's resolution lands
it correctly.

---

## 10. Interaction with pipeline recovery

- Scheduling creates the `Run` as the job's **first step**, before any config resolution — so
  a firing that fails partway still leaves a trace the recovery sweeper can act on. A firing
  that fails *before* `Run` creation (e.g. a bad config) is caught by startup validation
  (§11), not left silent.
- The recovery sweeper (`../solutions_v08.md`) owns interrupted scheduled runs — heartbeat
  detection, CAS ownership, resume-from-checkpoint. Scheduling adds nothing here beyond
  creating the `Run` and letting `requestRecovery(false)` keep Quartz out of it.
- `UNIQUE (report_type, frequency, scheduled_time)` on `Run` makes a repeated firing for the
  same slot fail fast at `Run` creation instead of after a full resolve pass.

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

## 12. OPEN decision — `EVERY_2_HOURS` / `EVERY_4_HOURS` window shape

Current production does **not** cover 00:00 → first fire for these two (first fire at
03:00 / 05:00 → window `01:00–03:00` / `01:00–05:00`; midnight to 01:00 is unreported each
day). CAMT054C's boundary frequencies *do* cover midnight (first window `00:00 → first
boundary`), so production is already inconsistent between the two families.

- **Match production** → `shape = ROLLING` for these two. Simpler: all four `EVERY_*` share
  one model, no boundary special-case.
- **Close the gap** → `shape = BOUNDARY` with an implicit leading `00:00`, giving first
  windows `00:00–03:00` / `00:00–05:00`.

Business query outstanding. The config's `shape` field makes flipping this a one-line change.

---

## 13. Testing on a fast cadence

For every frequency except `END_OF_DAY`, give the TEST profile a **dense spec** —
`interval = { first: 00:00, step: 2m, last: 23:58 }` or a dense `boundaries` list. The loader
generates a matching fast cron; every fire is a fresh window with fresh messages.
`cron-override` is the alternative for raw cron control, but firing faster than the real
boundaries only produces the first message per window (the pipeline dedups the rest via
`UQ_Outbox_Identity`).

**`END_OF_DAY` cannot be made to produce fresh reports on a fast cadence** while it stays
`CALENDAR_DAY` — its window is a function of the date alone, so repeat fires within a day are
pipeline no-ops. In order of preference:

1. **On-demand path** — an on-demand request with `period = <any date>` produces a fresh
   message for CAMT053S/053E/054D every call, exercising resolve → assemble → publish. Does
   not exercise scheduling's (trivial) previous-calendar-day arithmetic.
2. In TEST, point the `END_OF_DAY` schedule at `shape = BOUNDARY` with a dense `interval` —
   fast fresh windows through the scheduled path, at the cost of not testing the real
   calendar-day rule.

---

## 14. Pause / Resume — minimal, optional

Native Quartz, no new state:

- `scheduler.pauseTrigger(key)` / `resumeTrigger(key)` on the clustered scheduler; persisted
  in the JobStore, cluster-wide, survives restarts.
- A logical schedule may have more than one `TriggerKey` (`EVERY_30_MIN`); pause/resume for a
  `(report_type, frequency)` applies to **all** of them.
- Exposed through the same admin surface as §8, keyed by `(report_type, frequency)`.
- **Resume is forward-only** — firings missed while paused follow the §9 rule, not backfilled.
- Droppable from v1 — a pause is then a redeploy with the schedule removed from config.

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

---

## 16. Open decisions

- **§9 misfire policy** — deferred; recommendation recorded; needs a real Quartz test.
- **§12 `EVERY_2/4_HOURS` window shape** — pending a business answer.
- The eight fire times for CAMT054C `EIGHT_TIMES_PER_DAY` — a config value, TBD.
- DST gap/overlap policy (§4) — a proposal here; confirm with the business.
- Whether pause/resume (§14) ships in v1.
