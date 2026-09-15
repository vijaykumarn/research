# Commander Redesign — Architecture Options (v04)

This version folds in four rounds of review. Changes from v03 are marked **[v04]**.

---

## Foundation — settled baseline, shared by all options

**Quartz, clustered mode, JDBC JobStore in SQL Server** — guarantees exactly one pod fires a given scheduled trigger.

**Unit of work = one message.** A scheduled/on-demand/PHT trigger expands into N units of work: bundled config → one row per payment type (all-or-nothing); unbundled config → one row per account/alias; no-scope config → one config-only row.

**1. Unit-of-work creation is interleaved with resolution, not a step 0.**
For a scheduled run: page through `ReportConfig` (e.g., 500 at a time) → batch-resolve that page's scope/payment-type/account hierarchy → *now* the fan-out is known → insert the work-item rows for that page → move to the next page. "Create rows" and "resolve" are the same loop, one page at a time. This bounds memory — you never hold all thousands of rows' resolved data at once, just one page's worth.

**2. PHT gets a minted acceptance ID, same treatment as on-demand.**
`messageDate + messageTime + accountOwner` is not a safe outbound identity — a deliberate re-push landing in the same slot would be wrongly suppressed. When the PHT listener accepts a message off `CAMT.PHT.QUEUE`, it mints a Commander-side **PHT acceptance ID** (UUID), the same way on-demand mints an execution ID, and folds it into that message's outbound identity. Inbound redelivery dedup still uses `JMSMessageID`.

**3. Poison work items — attempt count and a real terminal state.**
`WorkItem` carries `attempt_count` and `last_error`. A configurable max (e.g., 5) moves a row to `FAILED_POISON` instead of retrying forever. `FAILED_POISON` is terminal and alertable — it pages/notifies and supports a manual redrive (operator resets it to `PENDING`, `attempt_count = 0`, after the underlying data issue is fixed).

**4. Spring Batch — stated position: not adopted, for stated reasons.**
- Its restart model resumes a **step**, not an arbitrary work item mid-page — less natural for "resume this specific config's unbundled fan-out that got half-published."
- It adds its own metadata schema (`BATCH_JOB_INSTANCE`, `BATCH_STEP_EXECUTION`, etc.) alongside the outbox/claim/tracking tables already needed — two persistence models for one concern.
- Commander's on-demand and PHT triggers are event-driven, not batch-launched — they don't map cleanly onto Spring Batch's single-job-instance-per-launch model.
- Worth revisiting later if Commander's needs grow toward large batch windows with complex chunk/skip/retry policies beyond what's described here — a reasonable *later* option, not a wrong one.

**5. Relay throughput ceiling — named, with a sharding fix.**
A single `sp_getapplock`-guarded relay pod is fine for latency but bottlenecks at thousands of outbox rows per run. Fix: shard the relay's claim — multiple pods claim batches of unsent outbox rows via `UPDATE ... SET claimed_by = @pod, claimed_at = SYSUTCDATETIME() WHERE status = 'PENDING' AND (claimed_by IS NULL OR claimed_at < stale_threshold)` with `READPAST` hints, or partition by `report_type` (6 types → up to 6 relay workers, one per queue, naturally parallel since each writes to a different MQ queue). No `sp_getapplock` singleton needed once claiming is row-level.

**Window computed from the trigger's scheduled fire time**, not wall-clock — stored on the `Run` record so a resumed run recomputes nothing and any dedup key stays stable across retries.

**Feature flags checked per report type / per config** before build and before publish. A false flag is a logged skip, marked `SKIPPED_FLAG_OFF` in the tracking row. **[v04]** This is terminal for the run it belongs to: if the flag flips back on later, the *next* scheduled trigger (or a fresh on-demand request) produces a new `Run`/`WorkItem` set — this run's skip decision is not retroactively revisited.

**Full config pass-through** — every `ReportConfig` field is mapped into `ReportMessage` in the assembly step, regardless of Commander's own logic.

**Cross-trigger claim on `(configId, window)`**, acquired per config as a scheduled run reaches it — an *optimization*, not the correctness guarantee (see Outbox below). On contention, on-demand/PHT NACK for redelivery with backoff; scheduled runs skip and let the next page/recovery pass retry.

---

## Recovery — shared design, scope corrected **[v04]**

- **Detection:** every `Run` has `started_at` and `heartbeat_at`, updated periodically by the owning pod while active. A **recovery sweeper** (its own clustered Quartz job, firing every N minutes) queries for stale runs.
- **[v04] Scope: scheduled runs only.** The sweeper queries `Run WHERE trigger_type = 'SCHEDULED' AND heartbeat_at < stale_threshold AND status != COMPLETED`. On-demand and PHT are **not** swept — a crash mid-processing is handled entirely by the broker redelivering the un-ACKed inbound message to another pod, which starts a *fresh* `Run` (fresh execution/acceptance ID) and redoes the small amount of work involved. This is safe because their `ProcessedInboundMessage` row and outbox row are written together on completion — a crash before completion means neither is recorded, so there is nothing for the sweeper to conflict with. Running the sweeper against on-demand/PHT as well would let broker redelivery and sweeper recovery race and mint two different `execution_id`s for the same logical work, defeating the outbox unique constraint and producing a duplicate.
- **Ownership arbitration:** the sweeper claims a stale run with a conditional update — `UPDATE Run SET owner_pod = @me, heartbeat_at = SYSUTCDATETIME() WHERE run_id = @id AND heartbeat_at = @lastSeenValue`. Only one of several simultaneously-restarting pods wins; losers move to the next stale run.
- **[v04] What resuming a scheduled run means — two parts:**
  1. Re-drive existing `WorkItem` rows where `status NOT IN (PUBLISHED, SKIPPED_FLAG_OFF, FAILED_POISON)`.
  2. **Resume paging** from `Run.last_config_id_processed` — re-enter the same resolve-and-create loop the original run used, picking up at the next page. Work items for configs beyond the last checkpoint were never created, so re-driving existing rows alone would silently drop the untouched tail of the run. Both parts run under the same claimed ownership.
- **[v04] Give-up path:** `Run.recovery_attempt_count` increments each time the sweeper recovers the same run. Past a bounded max, the sweeper sets `Run.status = ABANDONED` and alerts, rather than retrying indefinitely.
- **Where it runs:** a small dedicated clustered Quartz job, independent of the main trigger code paths.

---

## Table schemas **[v04 — revised]**

```sql
-- One row per run (scheduled fire, on-demand accept, PHT accept)
CREATE TABLE Run (
    run_id                   UNIQUEIDENTIFIER PRIMARY KEY,
    trigger_type             VARCHAR(20) NOT NULL,      -- SCHEDULED | ONDEMAND | PHT
    report_type              VARCHAR(20) NOT NULL,
    scheduled_time           DATETIME2 NOT NULL,        -- window derives from this, never wall-clock
    execution_id             UNIQUEIDENTIFIER NULL,     -- on-demand execution ID / PHT acceptance ID
    status                   VARCHAR(20) NOT NULL,      -- IN_PROGRESS | COMPLETED | ABANDONED
    owner_pod                VARCHAR(100) NOT NULL,
    started_at               DATETIME2 NOT NULL,
    heartbeat_at             DATETIME2 NOT NULL,
    last_config_id_processed BIGINT NULL,               -- [v04] paging checkpoint, scheduled only
    recovery_attempt_count   INT NOT NULL DEFAULT 0      -- [v04] drives ABANDONED
);

-- One row per unit of work (= one prospective message)
CREATE TABLE WorkItem (
    work_item_id     UNIQUEIDENTIFIER PRIMARY KEY,
    run_id           UNIQUEIDENTIFIER NOT NULL REFERENCES Run(run_id),
    config_id        BIGINT NOT NULL,
    scope_key        VARCHAR(200) NOT NULL,     -- payment type / account-or-none, per bundling rule
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    status           VARCHAR(20) NOT NULL,      -- PENDING | RESOLVED | PUBLISHED
                                                 -- | SKIPPED_FLAG_OFF | FAILED_POISON
                                                 -- [v04] ASSEMBLED dropped — see Option B note
    attempt_count    INT NOT NULL DEFAULT 0,
    last_error       NVARCHAR(MAX) NULL,
    updated_at       DATETIME2 NOT NULL
);

-- The actual correctness guarantee
-- [v04] logical_key replaced with typed columns — no delimiter/format-drift risk
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
                       -- sentinel for scheduled; real UUID for on-demand execution / PHT acceptance
    payload          NVARCHAR(MAX) NOT NULL,    -- serialized ReportMessage
    status           VARCHAR(20) NOT NULL,      -- PENDING | SENT
    claimed_by       VARCHAR(100) NULL,
    claimed_at       DATETIME2 NULL,
    created_at       DATETIME2 NOT NULL,
    CONSTRAINT UQ_Outbox_Identity UNIQUE
        (trigger_type, config_id, report_type, scope_key, window_start, window_end, execution_id)
);

-- Advisory claim, not correctness-critical
-- [v04] added TTL for crash release
CREATE TABLE ScopeClaim (
    config_id        BIGINT NOT NULL,
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    held_by_run_id   UNIQUEIDENTIFIER NOT NULL,
    acquired_at      DATETIME2 NOT NULL,
    expires_at       DATETIME2 NOT NULL,        -- [v04] acquired_at + fixed TTL (e.g. 10 min)
    PRIMARY KEY (config_id, window_start, window_end)
);

-- Inbound redelivery dedup (on-demand + PHT)
CREATE TABLE ProcessedInboundMessage (
    jms_message_id   VARCHAR(200) PRIMARY KEY,
    processed_at     DATETIME2 NOT NULL          -- retention-swept periodically
);
```

**[v04] `ScopeClaim` lifecycle, spelled out:**
- **Acquire:** `INSERT ... WHERE NOT EXISTS (SELECT 1 FROM ScopeClaim WHERE config_id=@c AND window_start=@ws AND window_end=@we AND expires_at > SYSUTCDATETIME())` — an expired row is treated as absent and can be overwritten.
- **Normal release:** delete the row once the work item reaches `PUBLISHED`, `SKIPPED_FLAG_OFF`, or `FAILED_POISON`.
- **Crash release:** nothing needs to act — TTL expiry *is* the release. No transfer logic needed; the claim is advisory, and the `Outbox` unique constraint remains the backstop even if two triggers briefly both believe they hold it near the TTL boundary.

**[v04] Outbox identity, per trigger** (typed columns, not a delimited string):
- Scheduled: `(SCHEDULED, configId, reportType, scopeKey, windowStart, windowEnd, sentinel-execution-id)`
- On-demand: `(ONDEMAND, configId, reportType, scopeKey, windowStart, windowEnd, executionId)`
- PHT: `(PHT, configId, reportType, scopeKey, windowStart, windowEnd, phtAcceptanceId)`

**[v04] "Outbox row already exists" is an explicit success path, not an error.** When a work item's outbox insert hits `UQ_Outbox_Identity`, that means the message was already durably recorded — by this item's own earlier attempt pre-crash, or by a racing duplicate producer. Handling: catch that specific constraint violation, `SELECT` the existing outbox row by its identity columns, and advance `WorkItem.status = PUBLISHED` regardless of which insert actually won. Only a *different* failure class (data error, MQ unavailable, etc.) increments `attempt_count` and can reach `FAILED_POISON`. This is a named branch in the pipeline's error handling, not an implicit assumption.

---

## The real fork: three options

**[v04]** Options A and B below are the same runtime shape (one deployable, one in-process pipeline per work item, outbox + relay) — they differ only in resumability granularity, not architecture. The genuine fork is **in-process pipeline vs. staged state machine**. A and B are kept as two rows because the resumability difference is a real, separately choosable cost/benefit — but neither should be read as a structurally distinct "architecture" from the other.

### Option A — In-process pipeline, redo-on-recovery

Each trigger's adapter creates `Run` + `WorkItem` rows page-by-page as it resolves. A single core pipeline then, per work item, in one pod, in-process: resolve → assemble → write outbox row (handling the "already exists" branch above) → advance `WorkItem.status = PUBLISHED`. No inter-stage persistence beyond that one status column. Recovery (scheduled-only, as above) re-picks any non-terminal `WorkItem` and re-runs it from scratch — safe because of the outbox unique constraint — and separately resumes paging from `last_config_id_processed`.

- ✅ Simplest runtime: one code path per item, easy to trace end to end.
- ✅ Lowest latency: no inter-stage sweep interval.
- ⚠️ Recovery granularity is "redo the whole work item." Fine since a work item is small, but a crash right before assembly finishes still costs a full re-resolve.
- ⚠️ If assembly is expensive for some report types, wholesale redo on recovery is wasted work.

### Option B — In-process pipeline with a resolve checkpoint (the pragmatic middle) **[v04 — simplified]**

Same as Option A, but `WorkItem.status` is used as a real mid-flight checkpoint: resolve → persist `status = RESOLVED` (+ the resolved data or a pointer to it) → assemble → write outbox row → advance `status = PUBLISHED`. **[v04]** The `ASSEMBLED` checkpoint from v03 is dropped — it was redundant, since the outbox row's existence already *is* the assembled-and-persisted marker. B's actual value is entirely the `RESOLVED` checkpoint: it skips re-resolution on recovery, which is the expensive step at volume (per the batched-resolution requirement).

- ✅ Skips re-resolution on recovery — the main cost Option A pays wastefully — without separate sweeper components or inter-stage latency in the happy path.
- ✅ Small added schema cost: resolved data needs a home between stages (extra nullable columns on `WorkItem`, or a side table).
- ⚠️ One extra persist per item (`RESOLVED`) versus Option A — negligible at this scale.
- ⚠️ Still one deployable, one pipeline — a resumability refinement of A, not a structural change.

### Option C — Staged pipeline, independent sweeper per stage

`Resolver`, `Assembler`, `Publisher` are separate Quartz-driven sweepers, each owning one state transition, running continuously and independently. Recovery is implicit for work items — a stale item is picked up by whichever sweeper owns its current state on the next tick. **[v04]** The scheduled-only recovery sweeper (paging-checkpoint + give-up path) still applies at the `Run` level, since paging resumption is a run-level concern regardless of how many stages the item-level pipeline has.

- ✅ Best operational visibility ("how many items stuck in RESOLVED") and independent per-stage scaling.
- ✅ Item-level recovery is the most natural of the three — the sweepers just keep running.
- ⚠️ Real added complexity: three sweeper components to build, deploy, monitor; more Quartz job configuration.
- ⚠️ Per-item latency is the sum of sweep intervals across stages unless stages trigger each other directly on completion — which, if done, makes it behave like Option B with extra ceremony.

---

## Comparison

| | A: In-process, redo-on-recovery | B: In-process, resolve-checkpointed | C: Staged sweepers |
|---|---|---|---|
| Build effort | Lowest | Medium | Highest |
| Recovery cost | Redoes whole item | Skips re-resolution | Resumes from last stage |
| Extra schema vs. foundation | None | Resolved-data columns/table | Same, plus per-stage sweeper config |
| Operational visibility | Coarse (item status only) | Coarse (item status only) | Fine-grained (per stage) |
| Components to run | 1 pipeline + 1 scheduled-only recovery sweeper | 1 pipeline + 1 scheduled-only recovery sweeper | 3 stage sweepers + 1 run-level recovery sweeper |
| Best fit if... | Resolve/assemble are cheap; redo-on-crash is a non-issue | You want to skip the expensive re-resolution step without extra moving parts | Volume/observability genuinely demands per-stage scaling and monitoring |

**Recommendation: Option B.** It gets the resumability the brief asks for — specifically skipping re-resolution, the step the batched-per-page requirement exists to make expensive-but-necessary — without three independently-deployed sweeper components. Option A is worth it only if resolve is confidently cheap enough that redoing it on every recovery is a non-issue. Option C is the right call only if per-stage operational visibility is a concrete, stated need, not a hedge against scale not yet reached.

---

## Open items for a next pass

- Sequence diagram for the FAQ Q2 race (scheduled + on-demand hitting the same `(config, window)` across two pods) worked through against Option B's final schema.
- `ScopeClaim` acquire/release code path in detail, including the TTL value and its relationship to expected per-config processing time.
