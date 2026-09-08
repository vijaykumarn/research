# Commander — Scheduling: Architecture Options (v01)

Three distinct approaches to how the **scheduled** trigger decides *when* each report runs and
*what window* that firing represents. This is a fresh design pass — it does not build on or
assume the earlier `scheduling.md` draft. Reflects `scheduling_goal.md` only.

---

## Settled foundation — applies to every option below

These aren't design choices between the options; any viable option needs all of them, and they
work the same way regardless of which option is picked.

**Window computation is a pure function, decoupled from triggering.**
`(frequency-definition, scheduled_time) → (window_start, window_end)` is one small,
independently unit-testable function. All three options below call the same function; they
differ only in *how they arrive at a `scheduled_time`* to feed it.

**DST policy — explicit, not left to defaults.**
- **Spring-forward gap** (a boundary time that doesn't exist that day): shift to the next
  instant that does exist. A skipped boundary would mean a permanently unreported slice of the
  day — worse than a slightly-shifted window one day a year.
- **Fall-back overlap** (a boundary time that occurs twice): always resolve to the *first*
  occurrence. Named, tested rule in the window function — not whatever the time library
  defaults to.
- `END_OF_DAY`'s previous-calendar-day window is legitimately 23 h or 25 h on a transition day
  — confirmed as intended (calendar day, not a fixed duration), not a bug to guard against.

**`Run` gets a uniqueness guarantee it doesn't currently have.**
The `Run` schema (`solutions_v08.md`) has no unique constraint on
`(report_type, frequency, scheduled_time)`. Today, a repeated firing for the same slot — a
misfire catch-up, a bug, a double-fire — produces two `Run` rows; each pages and re-resolves
the same configs before `UQ_Outbox_Identity` catches the actual duplicate at publish time. Add
`UNIQUE (report_type, frequency, scheduled_time)` on `Run` so a repeat fails fast at `Run`
creation instead of after a full resolve pass. This also satisfies goal requirement 5
(idempotency-friendly) at the scheduling layer specifically, rather than leaning entirely on
the pipeline's outbox constraint to catch it downstream.

**Startup validation.**
A startup runner diffs the distinct `(report_type, frequency)` values present in
`ReportConfig` (excluding `NEVER`) against whichever option's schedule definitions are
registered, and fails loud (or alerts) on any config value with no matching schedule.

---

## Option A — One Quartz trigger per `(report_type, frequency)`, generated cron

**Mechanism.** No grouping — `CAMT052B` + `EVERY_30_MIN` and `CAMT052BT` + `EVERY_30_MIN` are
two separate Quartz triggers even though they share a cadence; each `END_OF_DAY` report type
likewise gets its own trigger. Each cadence is authored **once** as a declarative spec
(start/end/step, or an explicit boundary list); cron expressions are *generated* from that spec
at startup rather than hand-written in parallel — closing the two-sources-of-truth risk
(requirement 7). The job body is minimal: compute the window from
`context.getScheduledFireTime()`, create exactly one `Run`.

**Requirement fit.**
- **Req 2** (one firing → one `Run` per report type) holds *by construction* — there is no
  fan-out step to get wrong.
- **Req 3** (misfire): native Quartz `withMisfireHandlingInstructionFireAndProceed`, one
  instruction per trigger, no cross-report-type coordination needed.
- **Req 8** (TEST fast-firing): swap the interval/boundary spec via profile — e.g.
  `EVERY_30_MIN` → a 30-second step in a test profile. Straightforward for every
  boundary-model frequency.

**Trade-offs.** More Quartz trigger registrations — roughly 20 once `EVERY_30_MIN`'s three
physical cron pieces per report type are counted (a single `CronTrigger` can't hold multiple
cron expressions, so "one logical cadence, three cron pieces" means three physical
`TriggerKey`s). Each trigger is independently pausable and independently recoverable, and
there is no fan-out logic to review for correctness. The `END_OF_DAY` leg of requirement 8
remains awkward here: the window is a function of the date alone, so firing the trigger faster
in TEST doesn't by itself produce a fresh window on each firing — this is true of every
Quartz-cron-based option (A and B), not specific to A.

---

## Option B — Grouped triggers with an explicit, idempotent fan-out job

**Mechanism.** One trigger per cadence *group*, matching the business language in the cadence
table directly — e.g. one trigger for "the three `END_OF_DAY` reports," one for "052B and
052BT every 30 minutes." The job, on firing, iterates its configured `report-types` list and
creates one `Run` per report type.

**Requirement fit.**
- **Req 2** is not free here and needs explicit handling: the fan-out step itself must be
  idempotent. If the pod dies after creating `Run`s for `CAMT053S` and `CAMT053E` but before
  `CAMT054D`, a retry must not re-create the first two. This is precisely what the
  foundation's `UNIQUE (report_type, frequency, scheduled_time)` constraint is for in this
  option specifically: each iteration attempts `Run` creation, treats "already exists" as
  success, and proceeds to the next report type. Without that constraint this option is
  unsafe; with it, safe and cheap.
- Fewer Quartz registrations, and the config shape reads the way a person would describe the
  schedule.

**Trade-offs.** Genuinely fewer moving parts in Quartz, at the cost of moving
correctness-critical logic (the idempotent fan-out) into application code instead of getting
it for free from "one trigger = one `Run`." Same `END_OF_DAY` TEST-speed limitation as Option
A, for the same reason. Worth picking only if Option A's trigger count is judged an actual
operational problem — 20 triggers is not large for a clustered Quartz JobStore, so this
trade-off should be justified by a concrete concern, not assumed.

---

## Option C — Tick-based, boundary-crossing decoupled from Quartz cron

**Mechanism.** Structurally different from A/B. Register one frequently-firing "heartbeat"
trigger per `(report_type, frequency)` (e.g. every minute) plus a persisted cursor —
`LastCompletedBoundary(report_type, frequency)`. On each tick, the job asks the boundary
function whether a new boundary has passed since the last completed one. If yes, produce a
`Run` for it (one at a time, per the fire-once-not-all-missed policy); if no, no-op.

**Requirement fit.**
- **Req 3** (misfire) stops being a Quartz-instruction concept — it becomes "compare a
  persisted cursor to current time," which is easier to unit test than Quartz misfire
  semantics and does not depend on `context.getScheduledFireTime()` landing exactly on a
  catalogued boundary (the exact-match fragility flagged in the earlier `scheduling.md`
  review).
- **Req 8** (TEST fast-firing) is cleanest here, including for `END_OF_DAY`: since the tick
  doesn't carry wall-clock cadence semantics itself, a test profile can tick every few seconds
  against an injected/accelerated `Clock`, producing a genuinely fresh window (and fresh
  messages) each time — this is the only option that actually solves the `END_OF_DAY` half of
  requirement 8 rather than leaving it open.
- **Req 2** again holds by construction if ticks are per report type.

**Trade-offs.** Gives up some of what Quartz cron gives for free. "Only one pod acts" still
holds — clustered Quartz still owns the heartbeat trigger's firing — but the
boundary-crossing detection and its cursor table become application-owned state that has to be
built and tested, effectively re-deriving something cron already does. More code and more
surface to review, in exchange for easier misfire testing and a real answer to the `END_OF_DAY`
TEST problem.

---

## Comparison

| | A — per-report-type cron | B — grouped + fan-out | C — tick-based |
|---|---|---|---|
| Req 2 (Run per report type) | Free, by construction | Needs idempotent fan-out (relies on foundation's `UNIQUE`) | Free, by construction |
| Req 3 (misfire correctness) | Quartz-native; exact-boundary-match care needed | Same as A | App-owned cursor; easier to test, no exact-match dependency |
| Req 7 (config maintainability) | Single spec → generated cron | Single spec → generated cron | Single spec, no cron at all |
| Req 8 (TEST fast-firing) | Easy for boundary freqs; `END_OF_DAY` stays awkward | Same as A | Easy for everything, including `END_OF_DAY` |
| New moving parts | Most Quartz registrations, least app logic | Fewer registrations, fan-out idempotency logic | Fewest registrations, most app-owned state (cursor table, tick logic) |

---

## Recommendation

**Option A.** It satisfies requirement 2 — the one hard, non-negotiable constraint in the brief
— structurally, rather than through fan-out logic that has to be gotten right and kept right.
It keeps Quartz doing what Quartz already does well (clustered firing, native misfire
instructions, native pause/resume for the optional requirement), and it's the option with the
least new machinery to design, build, and review.

Its one real weakness is the `END_OF_DAY` leg of requirement 8. I'd treat that as a smaller
problem than it looks: the `END_OF_DAY` message-production path can likely be exercised in
non-prod via the **on-demand** trigger (which already produces fresh messages for a supplied
config list on request) rather than requiring the *scheduled* path itself to fire quickly. If
that turns out not to be an acceptable substitute for whatever the TEST team actually needs,
Option C becomes the fallback — it's the only option that solves this requirement directly
rather than working around it — but Option C's added application-owned state should only be
taken on in exchange for a concretely stated testing need, not preemptively.

---

## Open items for the next pass

- Confirm the DST fall-back/spring-forward policy stated above with the business — it's a
  design proposal here, not a prior decision.
- Confirm whether the on-demand-trigger workaround for `END_OF_DAY` TEST-speed (under Option A)
  is actually sufficient for the TEST team's needs, or whether Option C's direct solution is
  required.
- Quartz trigger-count arithmetic under Option A once the exact boundary lists (including the
  still-undecided `EIGHT_TIMES_PER_DAY` times) are finalized.
- Misfire policy itself (fire-once-not-all, per report brief requirement 3) is a business
  decision to confirm, independent of which option is chosen to implement it.
- Pause/resume (optional): Option A/B get it natively via `scheduler.pauseTrigger` /
  `resumeTrigger`, persisted and cluster-wide with no new table. Option C would need the tick
  job itself to check a paused flag, since pausing the heartbeat trigger would also stop the
  cursor-advance check — worth a sentence if C is ever picked.
