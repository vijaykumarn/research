# Commander Redesign — Architecture Options (v08)

Eight rounds folded in. Changes from v07 are marked **[v08]**. Recommendation: **Option A**.
This pass carries one deliberate **removal** (the cross-trigger `ScopeClaim`), the scheduled
`execution_id` becoming **derived from `scheduled_time`** rather than a sentinel, a **`Run`
schema fix** (`frequency` + `window_start`/`window_end` columns + the scheduled slot-uniqueness
index the scheduling design already relies on), plus two small data-model clarifications.

---

## [v08] What changed and why

Three confirmed constraints from the product owner:

1. **A scheduled run and an on-demand request may both produce a message for the same config,
   report type, and window — and both should be delivered**, distinguished by trigger
   metadata. Neither suppresses the other.
2. **PHT recipients are disjoint from the scheduled and on-demand paths, enforced in data.**
   A PHT recipient has a `ReportConfig` row with `frequency = 'NEVER'`. The scheduled path
   selects work with `WHERE report_type = ? AND frequency = ? AND is_active = 1`, so a
   `NEVER` row is never picked up. The PHT flow resolves that config by recipient + report
   type directly, regardless of frequency.
3. **Executor accepts two semantically-equal messages** (same config, window, data) as long
   as trigger metadata distinguishes them.

Consequence: **the `ScopeClaim` table and its acquire/release logic are removed.** Scheduled
and on-demand messages already have distinct logical identities — a scheduled message's
`execution_id` is **derived from its `scheduled_time`** (see below), an on-demand message
carries its own minted UUID — so they never collide on `UQ_Outbox_Identity` and both publish
by design. `ScopeClaim` only ever made one trigger *wait* for the other, which is the opposite
of the confirmed intent, and it added a TTL, a `MERGE ... WITH (HOLDLOCK)` acquire, and the
"claim expires during recovery" wrinkle for no correctness benefit.

**[v08] Also settled this pass — the scheduled `execution_id` is derived, not a sentinel.**
Earlier drafts gave every scheduled-path outbox row the same all-zeros `execution_id`. That
made a scheduled `END_OF_DAY` firing and any *later* firing for the same window collapse to
one message — fine in production (it fires once a day) but it blocked fast-cadence testing,
where you want a fresh message each tick. Fix: a scheduled row's `execution_id` is
`uuid5("SCHEDULED|" + report_type + "|" + frequency + "|" + scheduled_time_iso)`. Every case
where dedup *must* hold already has a matching `scheduled_time` — crash recovery resumes the
same `Run`; `END_OF_DAY` and boundary-frequency misfire catch-ups snap `scheduled_time` to the
intended slot — so those still produce the same `execution_id` and still dedup. Genuinely
distinct firings (a fast-cadence test tick, or a manual re-run fired "now" with no
`scheduledTimeOverride`) get a distinct `execution_id` and a fresh message. Consistent with
on-demand/PHT, where `execution_id` also identifies "which execution produced this" — for
scheduled, the slot *is* that identity.

**What still guards against a genuine double-publish:** `UQ_Outbox_Identity` alone. It covers
the only case that matters — a scheduled run and its **own** crash-recovery (or a slow/paused
original pod still alive when recovery starts) both reaching the same item: same logical key,
one insert wins, the other is the named "already exists" success path. This is exactly how
Option A's redo-on-recovery model already works, so nothing new is needed.

---

## Foundation — settled baseline

**Quartz, clustered mode, JDBC JobStore in SQL Server** — one pod fires a given scheduled trigger.

**Unit of work = one message.** Bundled → one row per payment type (all-or-nothing); unbundled → one row per account/alias; no-scope → one config-only row.

**Interleaved paging + resolution.** Page through `ReportConfig` (e.g., 500 at a time) → batch-resolve the page's hierarchy → insert that page's `WorkItem` rows → advance the paging checkpoint → next page.

**[v08] Scheduled selection predicate.** The scheduled path fetches configs with
`report_type = ? AND frequency = ? AND is_active = 1`. A config with `frequency = 'NEVER'`
(the PHT-only marker) is therefore structurally excluded from every scheduled run — no
special-casing in code.

**Idempotent `WorkItem` creation.** `UNIQUE (run_id, config_id, scope_key, window_start, window_end)`; page inserts are idempotent (`MERGE`/`WHERE NOT EXISTS`); `Run.last_config_id_processed` advances in its own small transaction after a page's insert step completes, so a crash mid-page just re-runs a (now no-op) insert before advancing.

**Recovery reconciliation — resolved-fan-out is snapshotted, not re-derived blind.**
`WorkItem` rows, once created for a run, are the authoritative record of that run's fan-out —
recovery re-drives the existing set, it does not recompute membership.
- Recovery's batch-resolve step fetches only the *data* needed to build each existing
  `WorkItem`'s message — not whether that `WorkItem` should exist.
- If a re-resolve finds a `WorkItem`'s `scope_key` genuinely gone, the item moves to terminal
  `OBSOLETE` — not `FAILED_POISON`, not left `PENDING`. `OBSOLETE` is not alertable.
- **`OBSOLETE` requires a confirmed-absent result, not merely an empty one.** The resolve step
  returns a tri-state (found / confirmed-absent / query-failed). A transient DB blip or
  timeout is `query-failed` → ordinary retry path (`attempt_count`, eventually
  `FAILED_POISON`), never a silent `OBSOLETE`.
- Paging *forward* past `last_config_id_processed` still resolves fresh — there is no prior
  `WorkItem` set to reconcile against for the untouched tail of a run.

**PHT acceptance ID**, minted per pushed message, folded into outbound identity —
`messageDate/messageTime/accountOwner` alone isn't safe against a deliberate re-push.

**Scheduled `execution_id`**, derived deterministically as
`uuid5("SCHEDULED|{report_type}|{frequency}|{scheduled_time}")` — the same slot (recovery, a
snapped misfire catch-up) yields the same id and dedups; a genuinely distinct firing (a
fast-cadence test tick, an un-overridden manual re-run) yields a distinct id and a fresh
message. Not a sentinel.

**Poison items** — `attempt_count` + `last_error`; configurable max → terminal, alertable
`FAILED_POISON`, manual redrive supported. `OBSOLETE` is a sibling terminal state, explicitly
*not* alertable — legitimate data drift, not a failure.

**Spring Batch — not adopted** (step-level restart granularity, a second metadata schema,
event-driven triggers not fitting the batch-launch model).

**Relay throughput** — sharded outbox-row claiming or partitioned by `report_type`; no
single-pod bottleneck.

**Window from the trigger's scheduled fire time**, stored on `Run`, never wall-clock. (The
per-frequency window rules themselves are specified in `../scheduling/solution_v01.md`.)

**Feature flags** — checked per report type/config before build and publish; off →
`SKIPPED_FLAG_OFF`, terminal for that run.

**Full config pass-through**, plus **the message contract's own dedup fields**: `ReportMessage`
carries `(triggerType, configId, reportType, scopeKey, windowStart, windowEnd, executionId)`
explicitly, since the relay's claim→send→mark-`SENT` step can still duplicate at Executor and
none of these fields come from `ReportConfig` itself.

**[v08] No cross-trigger lock.** Scheduled, on-demand, and PHT run independently. Correctness
rests entirely on `UQ_Outbox_Identity`; there is no `(config, window)` claim.

**[v08 — proposed, pending confirm] On-demand guard for `NEVER` configs.** The on-demand path
takes an explicit list of config ids and does *not* filter by frequency. The product owner
guarantees no on-demand request will ever target a PHT-only recipient. As a cheap defensive
measure against a mistaken request, the on-demand path should **skip and log** any supplied
config whose `frequency = 'NEVER'` (record the skipped ids on the `Run`, do not fail the whole
request). If the product owner prefers to trust the guarantee with no check, this is dropped.

---

## Recovery — design

- **Detection:** clustered Quartz sweeper on stale `Run.heartbeat_at`.
- **Scope: scheduled only.** On-demand/PHT recover via broker redelivery into a fresh `Run` —
  sweeping them too would race redelivery against sweeper recovery and mint two execution IDs
  for one logical unit of work, defeating the outbox constraint.
- **Ownership:** CAS on `heartbeat_at`; one restarting pod wins a given stale run.
- **Resuming a scheduled run, two sub-cases:**
  1. **Existing non-terminal `WorkItem`s:** batch-resolve their pages' current data, redrive
     assembly/publish, reclassify any whose scope has genuinely vanished as `OBSOLETE`.
  2. **Unpaged tail** (`config_id > last_config_id_processed`): resolve and create fresh,
     exactly as normal processing does.
- **Give-up path:** `Run.recovery_attempt_count` bounded; past the max, `Run.status =
  ABANDONED`, alert.
- **Orphaned on-demand/PHT `Run` cleanup:** a separate low-frequency job marks long-stale
  non-scheduled `Run` rows `ABANDONED` for audit hygiene only — it does not attempt recovery.

---

## Table schemas

```sql
CREATE TABLE Run (
    run_id                   UNIQUEIDENTIFIER PRIMARY KEY,
    trigger_type             VARCHAR(20) NOT NULL,      -- SCHEDULED | ONDEMAND | PHT
    report_type              VARCHAR(20) NOT NULL,
    frequency                VARCHAR(20) NULL,          -- [v08] the schedule's frequency; NULL for ONDEMAND/PHT
    scheduled_time           DATETIME2 NOT NULL,
    window_start             DATETIME2 NOT NULL,        -- [v08] the run's reporting window, frozen at creation
    window_end               DATETIME2 NOT NULL,        --       (SCHEDULED: window function; ONDEMAND: request
                                                        --        period; PHT: parsed push timestamp)
    execution_id             UNIQUEIDENTIFIER NULL,
    status                   VARCHAR(20) NOT NULL,      -- IN_PROGRESS | COMPLETED | ABANDONED
    owner_pod                VARCHAR(100) NOT NULL,
    started_at               DATETIME2 NOT NULL,
    heartbeat_at             DATETIME2 NOT NULL,
    last_config_id_processed BIGINT NULL,
    recovery_attempt_count   INT NOT NULL DEFAULT 0,
    CONSTRAINT CK_Run_ExecutionId_Required CHECK (
        (trigger_type = 'SCHEDULED') OR (execution_id IS NOT NULL)
    ),
    CONSTRAINT CK_Run_Frequency_Required CHECK (
        (trigger_type <> 'SCHEDULED') OR (frequency IS NOT NULL)
    )
);

-- [v08] Slot uniqueness for scheduled runs only. A repeated firing for the same
-- (report_type, frequency, scheduled_time) — a misfire double-fire, a bug — fails fast at
-- Run creation instead of after a full resolve pass. Filtered to SCHEDULED so it never blocks
-- a legitimate on-demand re-request (which is deliberately a fresh Run, distinguished by
-- execution_id).
CREATE UNIQUE INDEX UQ_Run_ScheduledSlot
    ON Run (report_type, frequency, scheduled_time)
    WHERE trigger_type = 'SCHEDULED';

-- RESOLVED is written only if Option B's checkpoint is ever adopted; Option A (recommended)
-- does not use it. Left in the enum for forward compatibility.
CREATE TABLE WorkItem (
    work_item_id     UNIQUEIDENTIFIER PRIMARY KEY,
    run_id           UNIQUEIDENTIFIER NOT NULL REFERENCES Run(run_id),
    config_id        BIGINT NOT NULL,
    scope_key        VARCHAR(200) NOT NULL,
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    status           VARCHAR(20) NOT NULL,      -- PENDING | RESOLVED[Option B only] | PUBLISHED
                                                 -- | SKIPPED_FLAG_OFF | FAILED_POISON | OBSOLETE
    attempt_count    INT NOT NULL DEFAULT 0,
    last_error       NVARCHAR(MAX) NULL,
    updated_at       DATETIME2 NOT NULL,
    CONSTRAINT UQ_WorkItem_Identity UNIQUE
        (run_id, config_id, scope_key, window_start, window_end)
);

CREATE TABLE Outbox (
    outbox_id        UNIQUEIDENTIFIER PRIMARY KEY,
    trigger_type     VARCHAR(20) NOT NULL,
    config_id        BIGINT NOT NULL,
    report_type      VARCHAR(20) NOT NULL,
    scope_key        VARCHAR(200) NOT NULL,
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    execution_id     UNIQUEIDENTIFIER NOT NULL,
                       -- [v08] always set by the producing path, no sentinel:
                       --   on-demand -> minted per request; PHT -> minted per push;
                       --   scheduled -> uuid5("SCHEDULED|{report_type}|{frequency}|{scheduled_time}")
    payload          NVARCHAR(MAX) NOT NULL,    -- ReportMessage; must include the identity tuple
                                                 -- as message fields for Executor-side dedup
    status           VARCHAR(20) NOT NULL,      -- PENDING | SENT
    claimed_by       VARCHAR(100) NULL,
    claimed_at       DATETIME2 NULL,
    created_at       DATETIME2 NOT NULL,
    CONSTRAINT UQ_Outbox_Identity UNIQUE
        (trigger_type, config_id, report_type, scope_key, window_start, window_end, execution_id)
);

-- [v08] ScopeClaim removed — see "What changed and why" above.

CREATE TABLE ProcessedInboundMessage (
    jms_message_id   VARCHAR(200) PRIMARY KEY,
    processed_at     DATETIME2 NOT NULL
    -- retention floor must exceed the actual configured backout/redelivery-limit window on
    -- CAMT.ONDEMAND.QUEUE and CAMT.PHT.QUEUE; a redelivery arriving after this row is swept
    -- is treated as new and reprocessed — a known residual, bounded by that retention value.
);
```

**[v08] `Run` carries `frequency` and its reporting window.** Both are needed and were missing
from earlier DDL:
- `frequency` — a `report_type` can have several schedules (CAMT054C has three, all firing at
  21:00 among other times). Without it, the scheduled slot-uniqueness index would collapse
  three legitimate differently-windowed runs into one, and the recovery sweeper resuming a run
  past `last_config_id_processed` would not know which `frequency`'s config set to keep paging
  (the selection predicate is `report_type = ? AND frequency = ? AND is_active = 1`).
- `window_start` / `window_end` — frozen at `Run` creation so recovery's "keep paging" step
  stamps new `WorkItem`s without recomputing, and so a window-function change on a later deploy
  can't retroactively alter an in-flight run.
- `UQ_Run_ScheduledSlot` is a **filtered unique index on `SCHEDULED` rows only** — a plain
  table constraint would wrongly block a legitimate on-demand re-request (deliberately a fresh
  `Run`, distinguished by `execution_id`).

**`Run` creation hitting `UQ_Run_ScheduledSlot` is a named success path** — same posture as the
`Outbox` violation below. A repeated firing for a slot that already has a `Run` catches the
unique violation, logs, and **exits the job cleanly** — it does *not* throw a
`JobExecutionException` (which Quartz would treat as a failed execution and could escalate to
misfire/alert handling). The existing `Run` is either `COMPLETED` (idempotent no-op),
`IN_PROGRESS` (the recovery sweeper owns it), or `ABANDONED` (the give-up alert already fired;
re-running that slot is a manual action).

**`Outbox` "row already exists" is a named success path** — a `UQ_Outbox_Identity` violation
means the message is already durably recorded; `SELECT` the existing row and advance
`WorkItem.status = PUBLISHED` rather than treating it as an error. Only a *different* failure
class increments `attempt_count` toward `FAILED_POISON`.

**[v08] `execution_id` per trigger — the last column of `UQ_Outbox_Identity`:**

| Trigger | `execution_id` | Effect |
|---|---|---|
| Scheduled | `uuid5("SCHEDULED|{report_type}|{frequency}|{scheduled_time}")` | Same slot → same id → dedups (recovery, snapped misfire). Distinct firing → distinct id → fresh message. |
| On-demand | minted per accepted request | A re-request is a new message, never suppressed. |
| PHT | minted per accepted push | A re-push is a new message, never suppressed. |

Because a scheduled firing's `execution_id` follows `scheduled_time`, an `END_OF_DAY` schedule
run on a **fast cadence** (a dense `cron-override` firing every few minutes in TEST) produces
a **fresh message on every tick** — each tick has a distinct `scheduled_time`, hence a
distinct `execution_id`, hence a new `UQ_Outbox_Identity`. Production `END_OF_DAY` fires once a
day, so this changes nothing there; crash recovery and misfire catch-ups reuse the slot's
`scheduled_time` and still dedup.

**[v08] `WorkItem` terminal states no longer include a claim-release step.** A `WorkItem`
reaching `PUBLISHED` / `SKIPPED_FLAG_OFF` / `FAILED_POISON` / `OBSOLETE` simply updates its
own row — there is no `ScopeClaim` row to delete.

**Ordering — stated explicitly.** The sharded/partitioned relay gives no cross-message
ordering guarantee, by design. Safe here because every `ReportMessage` is self-contained (it
carries its own window and identity) and Executor is expected to process each message
independently. Confirm with the Executor team as an explicit contract point.

---

## Recommended architecture: Option A, in detail

Per work item, in one pod: resolve → assemble → write outbox row (handling the "already
exists" branch) → advance to `PUBLISHED`. No inter-stage persistence — `WorkItem.status` moves
`PENDING → PUBLISHED` (or a terminal alternative) in one step once assembly succeeds.

**Why A over B:** recovery batches resolution per page exactly as normal processing does, so
B's resolve-checkpoint no longer defends against a real cost. What A redoes on recovery is
assembly for still-incomplete items in an already-cheaply-resolved page. B remains available
if profiling later shows otherwise, at the price of its own resolved-data retention job.

**Why not C:** three independently deployed/monitored sweepers is real added operational
surface with no stated need behind it yet.

---

## Open items for the next (detailed-design) pass

- Concrete `ProcessedInboundMessage` retention — read off the actual
  backout-queue/max-redelivery configuration for `CAMT.ONDEMAND.QUEUE` and `CAMT.PHT.QUEUE`.
- Confirm the no-ordering-guarantee contract point, and the "two semantically-equal messages
  distinguished by trigger metadata" acceptance, with the Executor team.
- **[v08]** Confirm or drop the on-demand `frequency = 'NEVER'` skip-and-log guard.
- Scheduling design (trigger cadences, boundary windows, misfire policy, per-frequency
  reporting-window calculation, pause/resume) — separate document, `../scheduling/solution_v01.md`.
