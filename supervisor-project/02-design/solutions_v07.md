# Commander Redesign — Architecture Options (v07)

Seven rounds of review folded in. Changes from v06 are marked **[v07]**. Recommendation: **Option A**, detailed below with a reconciled recovery design. This is the final options-and-schema pass — remaining work is detailed design (concrete TTL/retention values, sequence diagrams), not further architecture decisions.

---

## Foundation — settled baseline, shared by all options

**Quartz, clustered mode, JDBC JobStore in SQL Server** — one pod fires a given scheduled trigger.

**Unit of work = one message.** Bundled → one row per payment type (all-or-nothing); unbundled → one row per account/alias; no-scope → one config-only row.

**Interleaved paging + resolution.** Page through `ReportConfig` (e.g., 500 at a time) → batch-resolve the page's hierarchy → insert that page's `WorkItem` rows → advance the paging checkpoint → next page.

**Idempotent `WorkItem` creation.** `UNIQUE (run_id, config_id, scope_key, window_start, window_end)`; page inserts are idempotent (`MERGE`/`WHERE NOT EXISTS`); `Run.last_config_id_processed` advances in its own small transaction after a page's insert step completes, so a crash mid-page just re-runs a (now no-op) insert before advancing.

**[v06] Recovery reconciliation — resolved-fan-out is snapshotted, not re-derived blind.**
v05 fixed *how* recovery batches resolution (per page, not per item) but not *what happens when re-resolution disagrees with the original run's fan-out* — a config's scope can legitimately change between the original attempt and recovery (account added/removed from scope). Re-deriving fresh each time either strands an orphaned `WorkItem` (scope shrank → its `scope_key` no longer appears → it sits `PENDING` forever → eventually a false `FAILED_POISON`) or silently expands the run beyond what "as of scheduled time" intends (scope grew → a new `WorkItem` appears that the original run never had).

Fix: `WorkItem` rows, once created for a run, are the authoritative record of that run's fan-out — recovery's job is to **re-drive the existing set**, not to recompute membership. Concretely:
- Recovery's batch-resolve step is used only to fetch the *data* needed to build the message for each existing `WorkItem` (account/alias/payment-type details) — not to decide *whether* that `WorkItem` should exist.
- If a re-resolve finds that a `WorkItem`'s `scope_key` genuinely no longer exists in the current data (e.g., account was removed from the config entirely), that item moves to a new terminal state `OBSOLETE` — not `FAILED_POISON` and not left `PENDING`. `OBSOLETE` is not alertable; it's logged and excluded from poison-item monitoring.
- **[v07] `OBSOLETE` requires a confirmed-absent result, not merely an empty one.** The batch-resolve call for a re-drive must distinguish "queried successfully and the scope_key is definitively gone" from "the query itself failed or timed out and returned nothing usable." Only the former retires the item as `OBSOLETE`. A transient DB blip or timeout during recovery's re-resolve must be treated as an ordinary resolve failure — increment `attempt_count`, leave the item eligible for retry (and eventually `FAILED_POISON` if it keeps failing) — never a silent `OBSOLETE`. Concretely: the resolve step returns a tri-state (found / confirmed-absent / query-failed), not a boolean "got data or not."
- Paging *forward* past `last_config_id_processed` (the "never got to this config at all yet" case) still resolves fresh, since there's no prior `WorkItem` set to reconcile against — this is unchanged from v05 and is the normal, correct behavior for the untouched tail of a run.
- This means recovery has two distinct sub-cases sharing one batching mechanism: (a) *re-drive existing items* — resolve their scope_keys' current data, mark genuinely-vanished ones `OBSOLETE`; (b) *continue paging* — resolve and create fresh, as normal processing does.

**PHT acceptance ID**, minted per pushed message, folded into outbound identity — `messageDate/messageTime/accountOwner` alone isn't safe against a deliberate re-push.

**Poison items** — `attempt_count` + `last_error`; configurable max → terminal, alertable `FAILED_POISON`, manual redrive supported. **[v06]** `OBSOLETE` (above) is a sibling terminal state, explicitly *not* alertable — it represents legitimate data drift, not a failure.

**Spring Batch — not adopted**, per the stated reasons in prior rounds (step-level restart granularity, a second metadata schema, event-driven triggers not fitting the batch-launch model).

**Relay throughput** — sharded outbox-row claiming or partitioned by `report_type`; no single-pod bottleneck.

**Window from the trigger's scheduled fire time**, stored on `Run`, never wall-clock.

**Feature flags** — checked per report type/config before build and publish; off → `SKIPPED_FLAG_OFF`, terminal for that run.

**Full config pass-through**, plus **the message contract's own dedup fields**: `ReportMessage` carries `(triggerType, configId, reportType, scopeKey, windowStart, windowEnd, executionId)` explicitly, since the relay's claim→send→mark-`SENT` step can still duplicate at Executor and none of these fields come from `ReportConfig` itself.

**Cross-trigger `(configId, window)` claim** — advisory; the `Outbox` unique constraint is the real guarantee.

---

## Recovery — final design

- **Detection:** clustered Quartz sweeper on stale `Run.heartbeat_at`.
- **Scope: scheduled only.** On-demand/PHT recover via broker redelivery into a fresh `Run` — sweeping them too would race redelivery against sweeper recovery and mint two execution IDs for one logical unit of work, defeating the outbox constraint.
- **Ownership:** CAS on `heartbeat_at`; one restarting pod wins a given stale run.
- **Resuming a scheduled run, two sub-cases (see reconciliation fix above):**
  1. **Existing non-terminal `WorkItem`s:** batch-resolve their pages' current data, redrive assembly/publish, reclassify any whose scope has genuinely vanished as `OBSOLETE`.
  2. **Unpaged tail** (`config_id > last_config_id_processed`): resolve and create fresh, exactly as normal processing does.
- **Give-up path:** `Run.recovery_attempt_count` bounded; past the max, `Run.status = ABANDONED`, alert.
- **Orphaned on-demand/PHT `Run` cleanup:** a separate low-frequency job marks long-stale non-scheduled `Run` rows `ABANDONED` for audit hygiene only — it does not attempt recovery.

---

## Table schemas

```sql
CREATE TABLE Run (
    run_id                   UNIQUEIDENTIFIER PRIMARY KEY,
    trigger_type             VARCHAR(20) NOT NULL,      -- SCHEDULED | ONDEMAND | PHT
    report_type              VARCHAR(20) NOT NULL,
    scheduled_time           DATETIME2 NOT NULL,
    execution_id             UNIQUEIDENTIFIER NULL,
    status                   VARCHAR(20) NOT NULL,      -- IN_PROGRESS | COMPLETED | ABANDONED
    owner_pod                VARCHAR(100) NOT NULL,
    started_at               DATETIME2 NOT NULL,
    heartbeat_at             DATETIME2 NOT NULL,
    last_config_id_processed BIGINT NULL,
    recovery_attempt_count   INT NOT NULL DEFAULT 0,
    -- [v06] enforce execution_id presence for non-scheduled triggers
    CONSTRAINT CK_Run_ExecutionId_Required CHECK (
        (trigger_type = 'SCHEDULED') OR (execution_id IS NOT NULL)
    )
);

-- [v06] RESOLVED is written only if Option B's checkpoint is ever adopted; Option A (recommended)
-- does not use it. Left in the enum for forward compatibility, not exercised by Option A's pipeline.
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
    execution_id     UNIQUEIDENTIFIER NOT NULL
                       DEFAULT '00000000-0000-0000-0000-000000000000',
    payload          NVARCHAR(MAX) NOT NULL,    -- ReportMessage; must include the identity tuple
                                                 -- as message fields for Executor-side dedup
    status           VARCHAR(20) NOT NULL,      -- PENDING | SENT
    claimed_by       VARCHAR(100) NULL,
    claimed_at       DATETIME2 NULL,
    created_at       DATETIME2 NOT NULL,
    CONSTRAINT UQ_Outbox_Identity UNIQUE
        (trigger_type, config_id, report_type, scope_key, window_start, window_end, execution_id)
);

CREATE TABLE ScopeClaim (
    config_id        BIGINT NOT NULL,
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    held_by_run_id   UNIQUEIDENTIFIER NOT NULL,
    acquired_at      DATETIME2 NOT NULL,
    expires_at       DATETIME2 NOT NULL,
    PRIMARY KEY (config_id, window_start, window_end)
);

CREATE TABLE ProcessedInboundMessage (
    jms_message_id   VARCHAR(200) PRIMARY KEY,
    processed_at     DATETIME2 NOT NULL
    -- retention floor must exceed the actual configured backout/redelivery-limit window on
    -- CAMT.ONDEMAND.QUEUE and CAMT.PHT.QUEUE; a redelivery arriving after this row is swept
    -- is treated as new and reprocessed — a known residual, bounded by that retention value.
);
```

**`Outbox` "row already exists" is a named success path** — a `UQ_Outbox_Identity` violation means the message is already durably recorded; `SELECT` the existing row and advance `WorkItem.status = PUBLISHED` rather than treating it as an error.

**[v06] `ScopeClaim` acquire, corrected.**
v05's `INSERT ... WHERE NOT EXISTS (... AND expires_at > now)` is wrong against a PK of `(config_id, window_start, window_end)`: when the existing row is expired, `NOT EXISTS` is satisfied and the `INSERT` is attempted anyway — but the row is still physically present, so it throws a primary-key violation rather than replacing it. Corrected acquire, as an atomic upsert:

**[v07] `HOLDLOCK` is required on the target.** Without it, SQL Server's `MERGE` is not safe under concurrent execution against the same key: two pods that both find the row absent (or both find it expired) can both take the same `WHEN` branch and race — resulting in a PK violation on the `INSERT` path, or a deadlock with SQL Server picking an arbitrary victim. This is not an edge case here — racing pods contending for the same `(config, window)` claim is the expected, normal situation this table exists to handle, so the lock hint is load-bearing, not defensive boilerplate.

```sql
MERGE ScopeClaim WITH (HOLDLOCK) AS target
USING (SELECT @config_id AS config_id, @window_start AS window_start, @window_end AS window_end) AS src
  ON target.config_id = src.config_id
 AND target.window_start = src.window_start
 AND target.window_end = src.window_end
WHEN MATCHED AND target.expires_at <= SYSUTCDATETIME() THEN
  UPDATE SET held_by_run_id = @run_id, acquired_at = SYSUTCDATETIME(), expires_at = @new_expiry
WHEN NOT MATCHED THEN
  INSERT (config_id, window_start, window_end, held_by_run_id, acquired_at, expires_at)
  VALUES (@config_id, @window_start, @window_end, @run_id, SYSUTCDATETIME(), @new_expiry);
```

A `MATCHED AND expires_at > now` case (still held, not expired) matches neither `WHEN` clause, so the `MERGE` is a no-op — the caller checks rows-affected and treats zero as "claim not acquired," which is the intended behavior. Release remains a plain `DELETE` on `PUBLISHED`/`SKIPPED_FLAG_OFF`/`FAILED_POISON`/`OBSOLETE`; crash release is still implicit TTL expiry.

**[v06] Ordering — stated explicitly.** The sharded/partitioned relay gives no cross-message ordering guarantee, by design. This is safe here because every `ReportMessage` is self-contained — it carries its own explicit window and identity, and Executor is expected to process each message independently rather than assuming arrival order reflects window order or config order. Worth confirming with the Executor team as an explicit contract point, not just an assumption on Commander's side.

---

## Recommended architecture: Option A, in detail

Per work item, in one pod: resolve → assemble → write outbox row (handling the "already exists" branch) → advance to `PUBLISHED`. No inter-stage persistence — `WorkItem.status` moves `PENDING → PUBLISHED` (or a terminal alternative) in one step once assembly succeeds.

**Why A over B (recap of the v05 flip, now final):** recovery batches resolution per page exactly as normal processing does, so B's resolve-checkpoint no longer defends against a real cost — the "expensive per-item re-resolution" B was built to avoid doesn't happen once recovery is fixed. What A redoes on recovery is assembly for still-incomplete items in an already-cheaply-resolved page, which is a much smaller and more defensible cost than v04's original framing assumed. B remains available if profiling later shows otherwise, at the price of its own resolved-data retention job.

**Why not C:** three independently deployed/monitored sweepers is real added operational surface with no stated need behind it yet; revisit if per-stage scaling or visibility becomes a concrete requirement.

---

## Open items for the next (detailed-design) pass

These are implementation-detail work, not further architecture questions:

- Sequence diagram for the FAQ Q2 race (scheduled + on-demand on the same `(config, window)` across two pods), walked through against this final schema and the corrected `MERGE`-based claim.
- Concrete `ScopeClaim` TTL — proposed starting point: comfortably above the observed p99 per-config processing time (needs a number from a load test, not a guess), reconsidered alongside the stated claim-expiry-during-recovery consequence.
- Concrete `ProcessedInboundMessage` retention — must be read off the actual backout-queue/max-redelivery configuration for `CAMT.ONDEMAND.QUEUE` and `CAMT.PHT.QUEUE`, not assumed.
- Confirm the no-ordering-guarantee contract point with the Executor team.
