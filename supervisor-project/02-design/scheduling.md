# Commander — Scheduling (v02)

How the **scheduled** trigger fires: which report type runs how often, on which days, and
what reporting window each firing covers. This is a separate concern from the message-production
pipeline (`how-it-works.md` / `solutions_v08.md`) but feeds directly into it.

**Scope of this document:** the trigger design only — cadences, boundary/window rules, wiring
to Quartz, the deploy-time config shape, misfire policy, and an optional pause/resume hook. Not
in scope: any richer schedule-management UI or runtime schedule editing.

v02 changes are summarised in §11.

---

## 1. What a firing produces

When a scheduled trigger fires, the pod that picks it up:

1. Takes the trigger's **scheduled** fire time (never wall-clock).
2. **Snaps it to a boundary** (Rule A) or applies the calendar-day rule (Rule B) — see §3 —
   producing `(scheduled_time, window_start, window_end)`, where `scheduled_time` is the
   *snapped* boundary, not the raw Quartz fire time.
3. Starts **one pipeline `Run` for exactly one report type**:

> `report_type` · `frequency` · `scheduled_time` · `window_start` · `window_end`

The pipeline then creates the `Run` and pages through
`ReportConfig WHERE report_type = ? AND frequency = ? AND is_active = 1`, exactly as
`how-it-works.md` section 4 describes.

A `Run` is single-report-type (`solutions_v08.md`). A firing therefore **always maps to one
`Run` for one report type** — never several. Where a config entry groups report types (§5),
the *loader* has already turned that into separate per-report-type triggers before anything
fires (§4).

---

## 2. Frequency catalogue

A `ReportConfig` carries a `frequency`. Each `(report_type, frequency)` pair is **one logical
schedule** — §4 covers how the loader turns that into Quartz jobs and triggers. All times are
in the single configured business timezone (a property, not hard-coded).

| Report type(s) | `frequency` | Fires at (business TZ) | Days | Reporting window |
|---|---|---|---|---|
| CAMT052B, CAMT052BT | `EVERY_30_MIN` | 00:30, 01:00, 01:30 … 21:00 | Mon–Fri | previous boundary → this fire (first fire: from 00:00) |
| CAMT052B, CAMT052BT | `EVERY_1_HOUR` | 01:00, 02:00 … 21:00 | Mon–Fri | previous boundary → this fire (first fire: from 00:00) |
| CAMT052B, CAMT052BT | `EVERY_2_HOURS` | 03:00, 05:00, 07:00 … 21:00 | Mon–Fri | previous boundary → this fire (first window 00:00→03:00 = 3 h) |
| CAMT052B, CAMT052BT | `EVERY_4_HOURS` | 05:00, 09:00, 13:00, 17:00, 21:00 | Mon–Fri | previous boundary → this fire (first window 00:00→05:00 = 5 h) |
| CAMT054C | `ONCE_PER_DAY` | 21:00 | Mon–Fri | 00:00 → 21:00, same day |
| CAMT054C | `FOUR_TIMES_PER_DAY` | 10:00, 13:00, 18:00, 21:00 | Mon–Fri | previous boundary → this fire (first fire: 00:00 → 10:00) |
| CAMT054C | `EIGHT_TIMES_PER_DAY` | 8 configurable times, last = 21:00 | Mon–Fri | previous boundary → this fire (first fire: 00:00 → first time) |
| CAMT053S, CAMT053E, CAMT054D | `END_OF_DAY` | 06:00 | Tue–Sat | the whole **previous calendar day**, 00:00 → 24:00 |
| any of the above | `NEVER` | never scheduled | — | — |

Notes:
- **`END_OF_DAY`** was renamed from `DAILY` for clarity — it sounded like a synonym of
  CAMT054C's once-a-day cadence (`ONCE_PER_DAY`) but has a different window rule (see `faq.md`
  Q6).
- **`NEVER`** is the PHT-only marker (see `solutions_v08.md`): the config exists so the PHT flow
  can resolve it, but no scheduled trigger ever selects it.
- **The eight times for `EIGHT_TIMES_PER_DAY` are not yet decided** — they come from config
  (see §5). The only fixed point is that the last one is 21:00.
- The 21:00 firing exists in all three CAMT054C cadences, but each reports a **different**
  window (00:00→21:00, 18:00→21:00, or *7th-boundary*→21:00). A config is on exactly one
  cadence (its `frequency`), so the three never collide — the paging query filters on
  `frequency`, so each firing sees a disjoint set of configs.

---

## 3. The two window rules

Only two rules cover everything.

### Rule A — boundary model (all frequencies except `END_OF_DAY`)

Each non-daily frequency is an **ordered list of daily boundary times**, with an implicit
`00:00` prepended. **This list is the authority** — the cron expressions in §4 are generated
from it, not maintained alongside it.

| `frequency` | Boundary list (with implicit leading 00:00) |
|---|---|
| `EVERY_30_MIN` | 00:00, 00:30, 01:00, … 20:30, 21:00 |
| `EVERY_1_HOUR` | 00:00, 01:00, 02:00, … 20:00, 21:00 |
| `EVERY_2_HOURS` | 00:00, 03:00, 05:00, 07:00, … 19:00, 21:00 |
| `EVERY_4_HOURS` | 00:00, 05:00, 09:00, 13:00, 17:00, 21:00 |
| `FOUR_TIMES_PER_DAY` | 00:00, 10:00, 13:00, 18:00, 21:00 |
| `EIGHT_TIMES_PER_DAY` | 00:00, *t₁ … t₇*, 21:00 |
| `ONCE_PER_DAY` | 00:00, 21:00 |

**Snap rule.** On firing, the job takes the trigger's scheduled fire time and finds the
boundary `b` in this list with the **largest boundary ≤ scheduled fire time**, within a small
tolerance (e.g. ±2 min). Then:

- If `b` is the implicit leading `00:00` → **skip** (00:00 is a window anchor, not a fire
  point) — this can only happen from an off-grid misfire, never a normal firing.
- If the scheduled fire time is not within tolerance of any boundary → **log and skip** (an
  off-grid time that can't be safely attributed to a boundary).
- Otherwise: `scheduled_time = b`, `window = (previous boundary, b]`. The first real boundary
  of the day therefore always yields a window starting at `00:00`.

Snapping (rather than requiring an exact match) is what makes a misfire catch-up fire land on
the right window — see §6.

*Worked examples:*
- `EVERY_2_HOURS`, firing at **03:00** → window **00:00 – 03:00**. Firing at **05:00** →
  **03:00 – 05:00**. Firing at **21:00** → **19:00 – 21:00**.
- `EVERY_4_HOURS`, firing at **05:00** → **00:00 – 05:00**. Firing at **09:00** → **05:00 – 09:00**.
- `FOUR_TIMES_PER_DAY`, firing at **10:00** → **00:00 – 10:00**. Firing at **13:00** →
  **10:00 – 13:00**.
- `ONCE_PER_DAY`, firing at **21:00** → **00:00 – 21:00**.
- A catch-up fire that Quartz scheduled for **13:24** (`EVERY_4_HOURS`) snaps to **13:00** →
  window **09:00 – 13:00**.

Consequence of anchoring the first window to midnight: for `EVERY_2_HOURS` and `EVERY_4_HOURS`
the day's **first** window is longer than the nominal interval (3 h and 5 h). This is
deliberate — it leaves no unreported gap between midnight and the first firing. Nothing after
21:00 is reported by any boundary-model frequency; the next day starts fresh at 00:00.

### Rule B — previous-calendar-day (only `END_OF_DAY`: CAMT053S, CAMT053E, CAMT054D)

The trigger fires at **06:00, Tuesday–Saturday**. The window is the **entire previous calendar
day**: `00:00:00` to `24:00:00` (business timezone) of the day before the firing.

- Tue 06:00 → covers Mon · Wed 06:00 → covers Tue · … · Sat 06:00 → covers Fri.
- Because it fires Tue–Sat only, **Saturdays and Sundays are never covered** — consistent with
  "business day = Mon–Fri".
- A misfired `END_OF_DAY` fire snaps to the intended **06:00 of its scheduled date**; the
  window is still the calendar day before that date.

### Daylight-saving

Boundary times are wall-clock local times in the business zone, resolved against the firing's
local date. On a DST transition day:

- **Spring-forward gap** (e.g. 02:00–03:00 does not exist): a boundary falling in the gap
  resolves to the **first valid instant after the gap**. That day's windows compress
  accordingly.
- **Fall-back overlap** (e.g. 01:00–02:00 occurs twice): a boundary in the overlap uses the
  **earlier** occurrence.
- **`END_OF_DAY`**: "previous calendar day 00:00 → 24:00" spans **23 or 25 elapsed hours** on a
  transition day. This is intended — it is a calendar day, not a fixed duration.

Java's default `ZonedDateTime` resolution already does gap-forward / overlap-earlier; this is
called out so it is a decision, not an accident.

---

## 4. Wiring to Quartz — the loader

The scheduler **loader** runs at startup and turns config into Quartz registrations. Two
principles (agreed):

**(a) Grouping is config sugar.** A config entry may list several `report-types`. The loader
**expands** it into one independent registration per report type. After startup there is no
"group" — only per-`(report_type, frequency)` schedules. Nothing at runtime fans out.

**(b) The boundary list is authoritative; crons are generated.** The loader derives the cron
expression(s) for each schedule from its `days` + `boundaries` (or interval spec). Cron
strings are never hand-maintained.

For each `(report_type, frequency)` the loader registers:

- **One `JobDetail`** carrying `report_type`, `frequency`, and the boundary list (or `END_OF_DAY`
  marker) in its `JobDataMap`.
- **One or more `CronTrigger`s** pointing at that job, each with an explicit
  `.inTimeZone(businessZone)` (Quartz defaults to the JVM zone otherwise). Most frequencies
  need one cron; `EVERY_30_MIN` needs two (`0 30 0-20 ? * MON-FRI` for the :30 fires and
  `0 0 1-21 ? * MON-FRI` for the :00 fires). Physical triggers for one schedule always have
  **disjoint fire times**.

Counts, once expanded:

| | Count |
|---|---|
| Config entries (with grouping) | ~8 (4 interval + 3 for CAMT054C + 1 `END_OF_DAY`) |
| Logical schedules `(report_type, frequency)` | **14** |
| Physical `CronTrigger`s | ~16 (14 + the extra one for each of `EVERY_30_MIN` × 2 report types) |
| Report types with **no** trigger | `NEVER` |

**Clustered JDBC JobStore** (already the pipeline's setup): only one pod fires a given trigger;
a pause is persisted and seen cluster-wide.

**The job, on firing:** apply the snap rule (§3 Rule A) or the calendar-day rule (§3 Rule B) to
`context.getScheduledFireTime()` → `(scheduled_time, window_start, window_end)` → start one
pipeline `Run` for `(report_type, frequency, scheduled_time, window_start, window_end)`. If a
non-`ABANDONED` `Run` already exists for this `(report_type, frequency, window)` today, skip
(an optimisation; the pipeline's `UQ_Outbox_Identity` would catch it anyway).

Illustrative generated crons (business TZ — informative, produced by the loader):

- `END_OF_DAY` group: `0 0 6 ? * TUE-SAT`
- CAMT054C `ONCE_PER_DAY`: `0 0 21 ? * MON-FRI`
- CAMT054C `FOUR_TIMES_PER_DAY`: `0 0 10,13,18,21 ? * MON-FRI`
- CAMT054C `EIGHT_TIMES_PER_DAY`: `0 0 <t1>,…,21 ? * MON-FRI`
- `EVERY_1_HOUR`: `0 0 1-21 ? * MON-FRI`
- `EVERY_2_HOURS`: `0 0 3-21/2 ? * MON-FRI`
- `EVERY_4_HOURS`: `0 0 5-21/4 ? * MON-FRI`
- `EVERY_30_MIN`: `0 30 0-20 ? * MON-FRI` + `0 0 1-21 ? * MON-FRI`

---

## 5. Configuration (deploy-time, via properties)

Fixed at deploy time, changed by redeploy. The entry carries only the **meaningful** inputs;
the loader derives the crons. Shape (illustrative):

```properties
commander.scheduling.timezone = <business zone>

# interval frequency — boundaries derived from an interval spec
commander.scheduling.triggers[0].report-types  = CAMT052B, CAMT052BT
commander.scheduling.triggers[0].frequency      = EVERY_2_HOURS
commander.scheduling.triggers[0].days           = MON-FRI
commander.scheduling.triggers[0].window-model   = BOUNDARY
commander.scheduling.triggers[0].interval       = { first: 03:00, step: 2h, last: 21:00 }

# N-times frequency — boundaries listed explicitly
commander.scheduling.triggers[5].report-types   = CAMT054C
commander.scheduling.triggers[5].frequency      = EIGHT_TIMES_PER_DAY
commander.scheduling.triggers[5].days           = MON-FRI
commander.scheduling.triggers[5].window-model   = BOUNDARY
commander.scheduling.triggers[5].boundaries     = <t1>, <t2>, ... , 21:00      # TBD

# end-of-day
commander.scheduling.triggers[7].report-types   = CAMT053S, CAMT053E, CAMT054D
commander.scheduling.triggers[7].frequency      = END_OF_DAY
commander.scheduling.triggers[7].days           = TUE-SAT
commander.scheduling.triggers[7].window-model   = PREVIOUS_CALENDAR_DAY
commander.scheduling.triggers[7].fire-at        = 06:00
```

There is **no hand-written `cron` field and no separate day list baked into a cron string** —
`days` is the single source of truth for day-of-week, `interval`/`boundaries` for the times.
Exact property layout is an implementation choice; the inputs are: report types · frequency ·
days · window model · times (interval spec or explicit boundary list; `fire-at` for
`END_OF_DAY`).

---

## 6. Misfire policy — OPEN (to be decided later)

Deferred by the product owner. Recommendation and mechanism recorded so the decision is easy.

**The question:** if every pod was down across a scheduled firing, what happens when the
cluster comes back?

**Mechanism (this part is not optional if any catch-up is wanted).**
`withMisfireHandlingInstructionFireAndProceed()` on a cron schedule is
`MISFIRE_INSTRUCTION_FIRE_ONCE_NOW` — the catch-up fire's raw scheduled time is **≈ recovery
time, not a boundary**. So the window can only be correct because of the **snap rule in §3**,
which snaps that off-grid time down to the most recent missed boundary. The earlier draft's
claim that "the window is correct because it comes from `getScheduledFireTime()`" was wrong
without the snap.

**Recommended policy:**
- Catch up the **most recent missed boundary only** — one `Run`, for the window
  `(previous boundary, most-recent-missed boundary]`. Do **not** replay every missed slot
  (down 6 h ≠ six catch-up runs).
- Older missed boundaries → manual **on-demand backfill**, not automatic.
- `END_OF_DAY`: catch up the most recent missed day only; older missed days → on-demand
  backfill.
- **Idempotency falls out for free:** the snapped catch-up window equals what the on-time fire
  would have produced, and the pipeline dedups on `window_start/window_end` (not
  `scheduled_time`), so `UQ_Outbox_Identity` transparently reconciles a catch-up against any
  partial on-time `Run`.
- Alternative if no catch-up is wanted at all: `MISFIRE_INSTRUCTION_DO_NOTHING` (skip the
  missed slot entirely, resume at the next boundary).

**Must be verified with a real Quartz test**, not just documented — confirm what
`getScheduledFireTime()` actually returns for a fire-once-now catch-up under a clustered
`JDBCJobStore`, and that the snap rule lands it on the intended boundary.

This interacts with §7 — a resumed trigger's first fire after a long pause is a misfire by the
same rule.

---

## 7. Pause / Resume — minimal, optional

Optional; keep it minimal. Quartz provides it natively:

- `scheduler.pauseTrigger(key)` / `resumeTrigger(key)` on the clustered scheduler. A pause is
  persisted in the JobStore — holds across restarts, seen cluster-wide.
- A logical schedule may have **more than one physical `TriggerKey`** (`EVERY_30_MIN`); a
  pause/resume for a `(report_type, frequency)` must apply to **all** of them.
- Expose it through whatever admin surface already exists (a small authenticated endpoint, or a
  custom Actuator endpoint) keyed by `(report_type, frequency)`.
- **No new table, no DB flag polling.**
- **Resume is forward-only** — firings missed while paused are handled by the §6 rule (catch up
  the most recent, don't replay the backlog), not backfilled.

If even this is more than wanted for v1, drop it — a pause is then achieved by redeploying with
the schedule removed from config.

---

## 8. Edge cases and notes

- **Business day = Mon–Fri, no holiday calendar** (for now). A weekday public holiday still
  runs its normal firings and reports that (likely near-empty) day; normal empty-report
  handling applies.
- **Daylight saving** — see §3: gap → forward, overlap → earlier, `END_OF_DAY` is a calendar
  day (23/25 h on a transition day, intended).
- **Every registered `CronTrigger` must set `.inTimeZone(businessZone)` explicitly** — Quartz
  defaults to the JVM zone otherwise, an easy miss in a multi-region deployment.
- **First window of the day is longer than the nominal interval** for `EVERY_2_HOURS` (3 h) and
  `EVERY_4_HOURS` (5 h), by the midnight-anchoring choice in §3 Rule A.
- **21:00–24:00 is never reported** by any boundary-model frequency.
- **`NEVER` configs** never appear in a scheduled `Run`.
- **Physical triggers for one logical schedule have disjoint fire times** — they never
  double-fire the same boundary.
- **Startup validation:** every distinct `(report_type, frequency)` in `ReportConfig` (other
  than `NEVER`) must have a matching registered schedule, or those configs would silently never
  be produced.

---

## 9. Open items

- The eight fire times for CAMT054C `EIGHT_TIMES_PER_DAY` (config value, TBD).
- **Misfire policy (§6)** — decision deferred; mechanism + recommendation recorded; **needs a
  real Quartz test** of fire-once-now scheduled-time behaviour before it's settled.
- Whether pause/resume (§7) is in v1 at all.

---

## 10. Handoff to the pipeline

Per firing, scheduling produces exactly:

`report_type` (one) · `frequency` · `scheduled_time` (the snapped boundary) ·
`window_start` · `window_end`

The pipeline (`solutions_v08.md`) takes it from there: one `Run`, paged config resolution,
`WorkItem` fan-out, Outbox, relay. Scheduling owns nothing after the `Run` is created.

---

## 11. Changes in v02

From the review of v01 against `goal.md` / `solutions_v08.md` / `how-it-works.md`:

1. **One firing → one `Run` for one report type**, stated explicitly (§1, §4). Report-type
   grouping is now **config sugar**: the loader expands each grouped entry into
   per-report-type schedules at startup; nothing fans out at runtime. Trigger arithmetic
   reconciled: ~8 config entries → 14 logical schedules → ~16 physical `CronTrigger`s.
2. **Boundary list is authoritative; crons are generated** by the loader from `days` +
   `boundaries`/`interval` (§3, §4, §5). No hand-written `cron` field, no separate day list —
   removes the drift risk between cron and boundary list.
3. **Snap rule added** (§3 Rule A): a firing's scheduled time is snapped to the nearest
   boundary ≤ it; off-grid or leading-`00:00` snaps are skipped. This is what makes a misfire
   catch-up land on the correct window.
4. **§6 misfire corrected**: `fireAndProceed` = fire-once-now, whose scheduled time is *not* a
   boundary — the v01 "window is still correct" claim was wrong without the snap rule. Policy
   restated as "catch up the most recent missed boundary only"; flagged as needing a real
   Quartz test.
5. **DST rule added** (§3): boundary times resolve gap-forward / overlap-earlier;
   `END_OF_DAY` is a calendar day (23/25 h on transition days, intended).
6. Nits: explicit `.inTimeZone(businessZone)` on every trigger (§4, §8); redundant `days` +
   cron-encoded-days removed (§5); disjoint-fire-time note for multi-trigger schedules (§8).
