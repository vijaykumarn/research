# Commander — Scheduling Design (v02)

The converged design. Supersedes the `scheduling_v01.md` options pass and the earlier
`../scheduling.md` draft. It picks **Option A** (one trigger per `(report_type, frequency)`,
generated crons, minimal job, window as a pure function), folds in the review findings, and
takes one idea from the bikili implementation while dropping bikili's per-boundary-trigger
mechanism at the product owner's request.

Scheduling decides *when* each report runs and *what window* a firing represents, then hands
off to the message-production pipeline (`../how-it-works.md` / `../solutions_v08.md`). It owns
nothing after a `Run` is created.

---

## 1. Settled foundation

Applies regardless of any remaining open item.

**Window computation is a pure function.**
`(frequency-spec, scheduled_time [, boundary-list]) → (window_start, window_end)` — one small,
independently unit-testable function. The trigger machinery's only job is to arrive at a
`scheduled_time` to feed it.

**`scheduled_time` is normalised to the boundary, never the raw fire time.**
For a boundary frequency the job resolves the firing to a catalogued boundary (§5); that
boundary's instant is `scheduled_time`. For a rolling frequency it is the scheduled fire
instant. For `END_OF_DAY` it is the 06:00 fire instant. This normalisation is what makes the
uniqueness constraint below work for misfire catch-ups.

**`Run` gets `UNIQUE (report_type, frequency, scheduled_time)`.**
`solutions_v08.md`'s `Run` has no such constraint today. Add it. A repeated firing for the same
slot — a misfire catch-up, a double-fire, a bug — then fails fast at `Run` creation instead of
doing a full config-resolve pass and only tripping `UQ_Outbox_Identity` downstream at publish.
Scheduling-layer idempotency, at the scheduling layer.

**DST policy — explicit, not left to the time library's defaults.**
- **Spring-forward gap** (a boundary time that does not exist that day): resolve to the next
  instant that does exist. A skipped boundary would leave a permanently unreported slice of
  the day — worse than a one-day-a-year shifted window.
- **Fall-back overlap** (a boundary time that occurs twice): resolve to the **first**
  occurrence.
- `END_OF_DAY`'s previous-calendar-day window spans **23 h or 25 h** on a transition day —
  intended (a calendar day, not a fixed duration), not a bug to guard against.

**Startup validation.**
A startup runner diffs the distinct `(report_type, frequency)` values in `ReportConfig`
(excluding `NEVER`) against the registered schedules, and fails loud on any config value with
no matching schedule — otherwise those configs would silently never be produced.

---

## 2. Frequency catalogue

A `ReportConfig.frequency` value selects which schedule picks it up
(`WHERE report_type = ? AND frequency = ? AND is_active = 1`). All times in the single
configured business timezone. Business day = Mon–Fri, no holiday calendar.

| Report type(s) | `frequency` | Shape | Fires | Days | Window |
|---|---|---|---|---|---|
| CAMT052B, CAMT052BT | `EVERY_30_MIN` | rolling | every 30 min, 00:30 → 21:00 | Mon–Fri | `fire − 30 min → fire` |
| CAMT052B, CAMT052BT | `EVERY_1_HOUR` | rolling | hourly, 01:00 → 21:00 | Mon–Fri | `fire − 1 h → fire` |
| CAMT052B, CAMT052BT | `EVERY_2_HOURS` | boundary | 03:00, 05:00 … 21:00 | Mon–Fri | previous boundary → this boundary; first = **00:00 → 03:00** |
| CAMT052B, CAMT052BT | `EVERY_4_HOURS` | boundary | 05:00, 09:00, 13:00, 17:00, 21:00 | Mon–Fri | previous boundary → this boundary; first = **00:00 → 05:00** |
| CAMT054C | `ONCE_PER_DAY` | boundary | 21:00 | Mon–Fri | 00:00 → 21:00 |
| CAMT054C | `FOUR_TIMES_PER_DAY` | boundary | 10:00, 13:00, 18:00, 21:00 | Mon–Fri | previous boundary → this; first = 00:00 → 10:00 |
| CAMT054C | `EIGHT_TIMES_PER_DAY` | boundary | 8 configurable times, last = 21:00 | Mon–Fri | previous boundary → this; first = 00:00 → first time |
| CAMT053S, CAMT053E, CAMT054D | `END_OF_DAY` | calendar-day | 06:00 | Tue–Sat | the whole **previous calendar day**, 00:00 → 24:00 |
| any of the above | `NEVER` | — | never scheduled | — | — |

- **`EVERY_30_MIN` / `EVERY_1_HOUR` are rolling**, not boundary: cheap, and rolling ==
  midnight-anchored there anyway (the first fire is exactly one interval past midnight).
- **`EVERY_2_HOURS` / `EVERY_4_HOURS` are boundary**, not rolling: this is how the first window
  of the day anchors to midnight (a rolling `EVERY_2_HOURS` would give 01:00 → 03:00). The
  boundary machinery's "sequence 0 = midnight → first boundary" rule gives the anchoring for
  free.
- **`NEVER`** — the PHT-only marker (`solutions_v08.md`); no trigger.
- The 21:00 firing exists in all three CAMT054C cadences with a different window each; a config
  is on exactly one cadence, and the paging query filters on `frequency`, so they never
  collide.

---

## 3. Trigger model

**One logical schedule per `(report_type, frequency)`.** Grouping in config is **sugar**: a
config entry may list several `report-types`; the **loader expands** it into one independent
registration per report type at startup. Nothing fans out at runtime — a firing is always one
`Run` for one report type.

**Crons are generated, never hand-authored alongside a boundary list.** Each schedule is
authored once as a declarative spec — an interval (`first` / `step` / `last`) or an explicit
boundary list, plus `days` — and the loader derives the cron expression(s). No two sources of
truth to drift (goal requirement 7).

**Boundary frequencies use a cron expression, not one trigger per boundary.** (This is the
deliberate change from bikili, which registers one `CronTrigger` per boundary time.) The
generated cron is:

| `frequency` | Generated cron (business TZ, `.inTimeZone(zone)` always set) |
|---|---|
| `EVERY_2_HOURS` | `0 0 3-21/2 ? * MON-FRI` |
| `EVERY_4_HOURS` | `0 0 5-21/4 ? * MON-FRI` |
| `ONCE_PER_DAY` | `0 0 21 ? * MON-FRI` |
| `FOUR_TIMES_PER_DAY` | `0 0 10,13,18,21 ? * MON-FRI` |
| `EIGHT_TIMES_PER_DAY` | `0 0 <t1>,…,21 ? * MON-FRI` (times TBD) |
| `EVERY_1_HOUR` | `0 0 1-21 ? * MON-FRI` |
| `EVERY_30_MIN` | `0 30 0-20 ? * MON-FRI` + `0 0 1-21 ? * MON-FRI` (two crons — no single cron hits exactly 00:30…21:00) |
| `END_OF_DAY` | `0 0 6 ? * TUE-SAT` |

A single cron works for a boundary frequency whenever its boundary times **share a minute**
(all `:00` in the current catalogue). If a future boundary sits on a different minute (e.g.
13:30), the loader emits **one cron per distinct minute value** — still far fewer than one per
boundary. `EVERY_30_MIN` is the only current case needing two.

**`JobDataMap`** per trigger carries `frequency` and, for boundary frequencies, the ordered
`boundaries` list. One `JobDetail` per `(report_type, frequency)`.

**Clustered JDBC JobStore** (already the pipeline's setup): only one pod fires a given trigger;
pauses are persisted cluster-wide.

Rough counts once the loader has expanded grouping: **~8 config entries → 14 logical schedules
→ ~16 physical `CronTrigger`s** (14 + the extra `EVERY_30_MIN` cron × 2 report types).

---

## 4. The window function

Pure, business-timezone, half-open `[start, end)`. Three shapes.

### Rolling — `EVERY_30_MIN`, `EVERY_1_HOUR`

`end = scheduled fire time`, `start = end − interval`. No boundary list, no sequence. Used only
where the first fire of the day is exactly one interval past midnight, so this coincides with
midnight-anchoring.

### Boundary — `EVERY_2_HOURS`, `EVERY_4_HOURS`, `ONCE_PER_DAY`, `FOUR_TIMES_PER_DAY`, `EIGHT_TIMES_PER_DAY`

The frequency's ordered boundary list, with an implicit `00:00` prepended:

| `frequency` | Boundary list (implicit leading 00:00) |
|---|---|
| `EVERY_2_HOURS` | 00:00, 03:00, 05:00, 07:00 … 21:00 |
| `EVERY_4_HOURS` | 00:00, 05:00, 09:00, 13:00, 17:00, 21:00 |
| `FOUR_TIMES_PER_DAY` | 00:00, 10:00, 13:00, 18:00, 21:00 |
| `EIGHT_TIMES_PER_DAY` | 00:00, *t₁ … t₇*, 21:00 |
| `ONCE_PER_DAY` | 00:00, 21:00 |

**Deriving the sequence** (since the single cron doesn't carry an index):

1. Take `context.getScheduledFireTime()`; convert to a local `(date, time)` in the business zone.
2. Find the largest real boundary `b` with `b ≤ time`.
   - **Normal firing:** `time` equals a boundary exactly → exact match, no fuzz.
   - **Off-grid firing** (a misfire whose scheduled time isn't a boundary — see §5): `b` is the
     most recent boundary before it. If `time` is not within a small tolerance of `b` *and*
     the raw gap is implausible, or `b` resolves to the implicit leading `00:00` → **log and
     skip**.
3. `window = (previous boundary, b]`, both resolved against `date`. The first real boundary of
   the day → `window_start = 00:00`.
4. `scheduled_time = b` on `date` (this, not the raw fire time, is stored on the `Run`).

*Examples:* `EVERY_2_HOURS` at 03:00 → `00:00–03:00`; at 05:00 → `03:00–05:00`; at 21:00 →
`19:00–21:00`. `EVERY_4_HOURS` at 05:00 → `00:00–05:00`. `FOUR_TIMES_PER_DAY` at 13:00 →
`10:00–13:00`. A catch-up fire Quartz scheduled for 13:24 (`EVERY_4_HOURS`) → snaps to 13:00 →
`09:00–13:00`, `scheduled_time = 13:00`.

### Calendar-day — `END_OF_DAY`

`scheduled_time` = the 06:00 fire instant. `window` = `00:00` to `24:00` (business zone) of the
**calendar day before** `scheduled_time`'s local date. A misfired `END_OF_DAY` fire uses its
*scheduled* 06:00 date, so the covered day is unchanged.

---

## 5. Misfire — OPEN (policy deferred by product owner)

Mechanism and recommendation recorded; the policy value is a business decision, tracked
separately.

`withMisfireHandlingInstructionFireAndProceed()` on a cron schedule =
`MISFIRE_INSTRUCTION_FIRE_ONCE_NOW` — the catch-up fire's raw scheduled time is **≈ recovery
time, not a boundary**. The window is nonetheless correct because §4's sequence derivation
**snaps that off-grid time down to the most recent boundary ≤ it**. There is no exact-match
requirement.

**Recommended policy:**
- Catch up the **most recent missed boundary only** — one `Run`, window `(previous boundary,
  most-recent-missed boundary]`. Do not replay every missed slot.
- Older missed boundaries → manual **on-demand backfill**.
- `END_OF_DAY`: catch up the most recent missed day only; older → on-demand backfill.
- **Idempotency is free:** the snapped catch-up `scheduled_time` and window equal what the
  on-time fire would have produced, so `UNIQUE (report_type, frequency, scheduled_time)` on
  `Run` — and `UQ_Outbox_Identity` downstream — reconcile a catch-up against any partial
  on-time run with no special handling.
- Alternative if no catch-up is wanted: `MISFIRE_INSTRUCTION_DO_NOTHING` (skip the slot,
  resume at the next boundary).

**Must be verified with a real Quartz test** under a clustered `JDBCJobStore` — confirm what
`getScheduledFireTime()` returns for a fire-once-now catch-up, and that the snap lands it on
the intended boundary.

---

## 6. Configuration (deploy-time, via properties)

Fixed at deploy time, changed by redeploy. The entry carries only the meaningful inputs; the
loader derives the crons. Illustrative:

```properties
commander.scheduling.timezone = <business zone>

# interval-spec form (EVERY_* frequencies)
commander.scheduling.triggers[0].report-types  = CAMT052B, CAMT052BT
commander.scheduling.triggers[0].frequency      = EVERY_2_HOURS
commander.scheduling.triggers[0].days           = MON-FRI
commander.scheduling.triggers[0].shape          = BOUNDARY            # ROLLING | BOUNDARY | CALENDAR_DAY
commander.scheduling.triggers[0].interval       = { first: 03:00, step: 2h, last: 21:00 }

# explicit boundary-list form (N-times frequencies)
commander.scheduling.triggers[4].report-types   = CAMT054C
commander.scheduling.triggers[4].frequency      = EIGHT_TIMES_PER_DAY
commander.scheduling.triggers[4].days           = MON-FRI
commander.scheduling.triggers[4].shape          = BOUNDARY
commander.scheduling.triggers[4].boundaries     = <t1>, <t2>, ... , 21:00     # TBD

# calendar-day
commander.scheduling.triggers[7].report-types   = CAMT053S, CAMT053E, CAMT054D
commander.scheduling.triggers[7].frequency      = END_OF_DAY
commander.scheduling.triggers[7].days           = TUE-SAT
commander.scheduling.triggers[7].shape          = CALENDAR_DAY
commander.scheduling.triggers[7].fire-at        = 06:00

# optional — TEST only: override the generated wake-up schedule with a raw cron
commander.scheduling.triggers[0].cron-override  = 0 0/2 * ? * MON-FRI
```

- **No hand-written `cron` field in normal use**, no separate day list baked into a cron
  string. `days` is the single source of truth for day-of-week; `interval` / `boundaries` for
  the times; `fire-at` for `CALENDAR_DAY`. The loader generates the cron(s).
- `interval` and `boundaries` are interchangeable for any `BOUNDARY` shape — use whichever is
  less to type.
- **`cron-override`** (optional, TEST): when present the loader registers it verbatim (still
  `.inTimeZone(zone)`) instead of generating. The boundary list still governs the window via
  §4's snap; the off-grid "skip" guard is disabled under an override (off-grid is expected
  there). Absent in production, where generation is the single source of truth.

---

## 7. Testing on a fast cadence

For every frequency except `END_OF_DAY`, exercise the scheduled path quickly by giving the
TEST profile a **dense spec** — `interval = { first: 00:00, step: 2m, last: 23:58 }` or a dense
`boundaries` list. The loader generates a matching fast cron; every fire is a fresh window with
fresh messages. `cron-override` is the alternative if you want raw cron control, but firing
faster than the real boundaries only produces the first message per window (the pipeline dedups
the rest via `UQ_Outbox_Identity`).

**`END_OF_DAY` cannot be made to produce fresh reports on a fast cadence** while it stays
`CALENDAR_DAY` — its window is a function of the date alone, so repeat fires within a day are
pipeline no-ops. Options, in order of preference:

1. **On-demand path** — an on-demand request with `period = <any date>` produces a fresh
   message for CAMT053S/053E/054D every call, exercising resolve → assemble → publish. It does
   not exercise scheduling's (trivial) previous-calendar-day arithmetic.
2. In TEST, point the `END_OF_DAY` schedule at `shape = BOUNDARY` with a dense `interval` — you
   get fast fresh windows through the *scheduled* path, at the cost of not testing the real
   calendar-day rule.
3. If neither is acceptable to the TEST team, the **tick-based Option C** from
   `scheduling_v01.md` is the only design that fires `END_OF_DAY` fast through the scheduled
   path directly — but its cursor table and app-owned boundary-crossing logic should be taken
   on only against a concrete stated need, not preemptively.

---

## 8. Pause / Resume — minimal, optional

Native Quartz, no new state:

- `scheduler.pauseTrigger(key)` / `resumeTrigger(key)` on the clustered scheduler; persisted in
  the JobStore, cluster-wide, survives restarts.
- A logical schedule may have more than one `TriggerKey` (`EVERY_30_MIN`); pause/resume for a
  `(report_type, frequency)` applies to **all** of them.
- Exposed through the existing admin surface (a small authenticated endpoint or a custom
  Actuator endpoint), keyed by `(report_type, frequency)`.
- **Resume is forward-only** — firings missed while paused follow the §5 rule (catch up the
  most recent, don't replay), not backfilled.
- Droppable from v1 — a pause is then a redeploy with the schedule removed from config.

---

## 9. Edge cases

- **Business day = Mon–Fri, no holiday calendar** (for now). A weekday holiday still runs its
  firings and reports that (near-empty) day; normal empty-report handling applies.
- **Daylight saving** — §1: gap → forward, overlap → earlier, `END_OF_DAY` is a calendar day
  (23/25 h on transition days, intended).
- **Every registered `CronTrigger` sets `.inTimeZone(businessZone)` explicitly** — Quartz
  defaults to the JVM zone otherwise.
- **First window of the day is longer than the nominal interval** for `EVERY_2_HOURS` (3 h) and
  `EVERY_4_HOURS` (5 h) — the midnight-anchoring in §4.
- **21:00 → 24:00 is never reported** by any boundary or rolling frequency.
- **`NEVER` configs** never appear in a scheduled `Run`.
- **A firing whose scheduled time can't be attributed to a boundary** (implausible off-grid,
  or snaps to the implicit `00:00`) is logged and skipped, not forced onto a wrong window.

---

## 10. Handoff to the pipeline

Per firing, scheduling produces exactly:

`report_type` (one) · `frequency` · `scheduled_time` (the resolved boundary / fire instant) ·
`window_start` · `window_end`

The pipeline (`../solutions_v08.md`) creates one `Run` — now guarded by
`UNIQUE (report_type, frequency, scheduled_time)` — and takes it from there.

---

## 11. Open items

- The eight fire times for CAMT054C `EIGHT_TIMES_PER_DAY` (config value, TBD). bikili's
  production value was `03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00` — a data point,
  not a decision.
- **Misfire policy (§5)** — deferred; mechanism + recommendation recorded; needs a real Quartz
  test of fire-once-now `getScheduledFireTime()` behaviour before it is settled.
- Whether the `END_OF_DAY` fast-TEST need (§7) requires anything beyond the on-demand
  workaround — and therefore whether Option C is ever built.
- Confirm the DST gap/overlap policy (§1) with the business — a proposal here, not a prior
  decision.
- Whether pause/resume (§8) ships in v1.

---

## 12. What changed from v01 / the bikili look

- **Picked Option A** from `scheduling_v01.md` (per-`(report_type, frequency)` trigger,
  generated crons, pure window function, minimal job).
- **Adopted** the v01 pass's `UNIQUE (report_type, frequency, scheduled_time)` on `Run` and its
  explicit DST policy.
- **From bikili:** the window function as a pure `(frequency, scheduledFireTime,
  boundaries) → period` calculator, and the "sequence 0 = midnight → first boundary" rule that
  gives `EVERY_2_HOURS` / `EVERY_4_HOURS` their midnight-anchored first window when modelled as
  boundary frequencies.
- **Rejected from bikili** (product-owner decision): one `CronTrigger` per boundary time.
  Boundary frequencies use a single generated cron (or one per distinct minute value); the job
  derives the sequence by matching / snapping `getScheduledFireTime()` against the boundary
  list — exact on a normal firing, snap-to-nearest on a misfire.
- **Rejected from bikili:** rolling look-back for `EVERY_2_HOURS` / `EVERY_4_HOURS` (would
  break the chosen midnight-anchoring); those are boundary frequencies here.
- Retained from the earlier `../scheduling.md` draft: grouping-as-config-sugar with loader
  expansion, generated-not-authored crons, explicit `.inTimeZone`, the `cron-override` and
  dense-spec TEST affordances, minimal native pause/resume.
