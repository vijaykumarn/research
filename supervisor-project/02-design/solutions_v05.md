# Commander Redesign — Architecture Options (v05)

This version folds in five rounds of review. Changes from v04 are marked **[v05]**.

---

## Foundation — settled baseline, shared by all options

**Quartz, clustered mode, JDBC JobStore in SQL Server** — guarantees exactly one pod fires a given scheduled trigger.

**Unit of work = one message.** Bundled config → one row per payment type (all-or-nothing); unbundled config → one row per account/alias; no-scope config → one config-only row.

**Unit-of-work creation is interleaved with resolution, not a step 0.** For a scheduled run: page through `ReportConfig` (e.g., 500 at a time) → batch-resolve that page's hierarchy → insert that page's `WorkItem` rows → advance the paging checkpoint → next page. Bounds memory to one page's resolved data at a time.

**[v05] `WorkItem` creation is idempotent — a natural-key unique constraint, not a page-sized transaction.**
The v04 doc left open what happens if a pod crashes mid-page (some rows inserted, some not) and didn't commit to a transaction boundary. Resolved: `WorkItem` gets `UNIQUE (run_id, config_id, scope_key, window_start, window_end)`, and page creation inserts idempotently (`MERGE` / `INSERT ... WHERE NOT EXISTS` per row or in a batch statement) rather than relying on one all-or-nothing transaction across a page's rows. Consequence: a partial page is safe to resolve again — re-inserting the page's rows is a no-op for whatever already landed, and only the missing rows get created. `Run.last_config_id_processed` is advanced only after the page's idempotent insert step reports complete, in its own small transaction — so a crash between "rows inserted" and "checkpoint advanced" simply re-does that page's (now no-op) insert and then advances the checkpoint. This avoids both the multi-thousand-row single transaction and the duplicate-row risk of a bare UUID PK with no natural key.

**PHT gets a minted acceptance ID**, same treatment as on-demand — folded into that message's outbound identity, since `messageDate + messageTime + accountOwner` isn't safe (a deliberate re-push in the same slot would be wrongly suppressed).

**Poison work items** — `attempt_count` + `last_error` on `WorkItem`; a configurable max moves a row to terminal, alertable `FAILED_POISON`, with manual redrive support.

**Spring Batch — not adopted.** Its restart model resumes a step, not an arbitrary mid-page work item; it adds a second metadata schema alongside the outbox/claim/tracking tables already needed; on-demand/PHT are event-driven, not batch-launched. Worth revisiting only if Commander's needs grow toward large, complex chunk/skip/retry batch windows beyond what's described here.

**Relay throughput** — sharded by claiming batches of unsent outbox rows (`READPAST` hints) or partitioned by `report_type` (up to 6 parallel relay workers, one per queue). No single-pod `sp_getapplock` bottleneck.

**Window computed from the trigger's scheduled fire time**, stored on `Run`, never wall-clock — keeps the dedup key stable across retries.

**Feature flags** checked per report type / per config before build and publish. A false flag → `SKIPPED_FLAG_OFF`, terminal for that run; a later flag flip is picked up by the *next* run, not retroactively.

**Full config pass-through** — every `ReportConfig` field maps into `ReportMessage` in assembly.

**[v05] The message contract also needs fields `ReportConfig` doesn't provide.**
The outbox unique constraint prevents a duplicate *row in the outbox*, but the relay's own claim → send-to-MQ → mark-`SENT` step has the same crash-window problem as everything else here: a crash between "sent to MQ" and "marked SENT" causes a re-send. So **Executor receives the stream at-least-once, not exactly-once**, and must dedupe on the same identity Commander uses internally. That means `ReportMessage` must explicitly carry `(triggerType, configId, reportType, scopeKey, windowStart, windowEnd, executionId)` as message fields — none of which come from `ReportConfig`, so "full config pass-through" does not cover them. This is a message-contract requirement, stated here so it isn't discovered downstream when Executor asks "how do I tell these two messages are the same report."

**Cross-trigger claim on `(configId, window)`** — advisory only; the `Outbox` unique constraint is the actual guarantee. On contention, on-demand/PHT NACK-and-retry with backoff; scheduled work skips and lets the next pass retry.

---

## Recovery — shared design

- **Detection:** every `Run` has `heartbeat_at`; a dedicated clustered Quartz sweeper finds stale ones.
- **Scope: scheduled runs only.** On-demand/PHT recover via broker redelivery alone — a crash before completion means neither `ProcessedInboundMessage` nor the outbox row was written, so there's nothing for a sweeper to conflict with. Sweeping them as well would let redelivery and sweeper-recovery race, mint two different execution IDs for the same logical work, and defeat the outbox constraint.
- **Ownership arbitration:** compare-and-swap on `heartbeat_at` — only one restarting pod wins a given stale run.
- **[v05] Resuming a scheduled run re-pages in batches — it does not re-resolve item by item.**
  v04's recovery re-drove non-terminal `WorkItem` rows individually, which silently reintroduced the exact per-config resolution the batched-per-page requirement exists to prevent — recovering a run with 3,000 non-terminal items would do 3,000 individual resolutions. Corrected: recovery identifies the **distinct pages** touched by non-terminal items (grouping by the page boundary those `config_id`s fall into), batch-resolves each affected page in one pass (same query shape as normal processing), and then re-drives that page's non-terminal items using the freshly-resolved data. Paging resumption past `last_config_id_processed` uses this same batched loop, so "drain existing non-terminal items" and "keep paging forward" are now the same mechanism applied to different page ranges, not two different code paths.
  - **This changes the Option A/B trade-off — see below.**
- **Give-up path:** `Run.recovery_attempt_count` bounds retries; past the max, `Run.status = ABANDONED` and alerts.
- **[v05] Orphaned on-demand/PHT `Run` rows.** Since these aren't swept for recovery, a crash mid-processing leaves their `Run` row stuck `IN_PROGRESS` forever (the actual retry happens via redelivery into a *new* `Run`). Fix: a separate, low-frequency cleanup job marks `Run` rows with `trigger_type != 'SCHEDULED'` and `heartbeat_at` older than a generous threshold (comfortably past any realistic redelivery window) as `ABANDONED` — for audit/reporting hygiene only, it does not attempt recovery.

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
    last_config_id_processed BIGINT NULL,               -- paging checkpoint, scheduled only
    recovery_attempt_count   INT NOT NULL DEFAULT 0
);

-- [v05] added UQ_WorkItem_Identity for idempotent page creation
CREATE TABLE WorkItem (
    work_item_id     UNIQUEIDENTIFIER PRIMARY KEY,
    run_id           UNIQUEIDENTIFIER NOT NULL REFERENCES Run(run_id),
    config_id        BIGINT NOT NULL,
    scope_key        VARCHAR(200) NOT NULL,
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    status           VARCHAR(20) NOT NULL,      -- PENDING | RESOLVED | PUBLISHED
                                                 -- | SKIPPED_FLAG_OFF | FAILED_POISON
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
    payload          NVARCHAR(MAX) NOT NULL,    -- serialized ReportMessage; [v05] must include
                                                 -- triggerType/configId/reportType/scopeKey/
                                                 -- windowStart/windowEnd/executionId as message
                                                 -- fields, so Executor can dedupe at-least-once delivery
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
    -- [v05] retention floor: must exceed the maximum possible broker redelivery / backout-queue
    -- window for CAMT.ONDEMAND.QUEUE and CAMT.PHT.QUEUE. If a redelivery arrives after this row
    -- has been swept, it is treated as new — a fresh execution_id is minted and it is reprocessed
    -- and republished. This is a known residual risk, not eliminated by this design; the retention
    -- value must be set from the actual configured backout/retry policy on those queues, not a guess.
);
```

**`Outbox` "row already exists" is a named success path.** A `UQ_Outbox_Identity` violation on insert means the message is already durably recorded — by this item's own pre-crash attempt, or a racing duplicate producer. Handling: catch that specific violation, `SELECT` the existing row, advance `WorkItem.status = PUBLISHED`. Only a different failure class increments `attempt_count` toward `FAILED_POISON`.

**`ScopeClaim` lifecycle:** acquire via `INSERT ... WHERE NOT EXISTS (... AND expires_at > SYSUTCDATETIME())`; release by deleting the row on `PUBLISHED`/`SKIPPED_FLAG_OFF`/`FAILED_POISON`; crash release is implicit TTL expiry, no transfer logic needed since the claim is advisory. **[v05] Stated consequence:** if a scheduled run sits stale long enough for recovery detection to kick in, its held claims expire first — widening the window in which an on-demand request targeting the same `(config, window)` proceeds and produces independently. This is accepted by design (on-demand must never be suppressed), not a defect, but is a visible effect worth knowing about when interpreting duplicate-effort metrics.

---

## The real fork: two comparable shapes, one genuine question

Options A and B are the same runtime architecture (one deployable, one in-process pipeline per work item, outbox + relay); they differ only in recovery granularity. Option C is the actual structural alternative (staged, independently-scaled sweepers).

### Option A — In-process pipeline, redo-on-recovery

Per work item, in one pod: resolve → assemble → write outbox row (handling the "already exists" branch) → advance to `PUBLISHED`. No inter-stage persistence. **[v05]** Recovery no longer means "re-resolve this one item" — per the corrected recovery design, it means "batch-resolve the affected page, then redo assembly for that page's non-terminal items." This is materially cheaper than v04's per-item framing: the expensive part (resolution) is batched exactly as it is in normal processing, and only assembly is redone per item.

- ✅ Simplest runtime, easiest to trace end to end.
- ✅ Lowest latency, no inter-stage persistence.
- ✅ **[v05]** With batched-page recovery, the "redo-on-recovery" cost is no longer an N-individual-resolutions problem — it's "re-run one page's batch resolve, redo assembly for its incomplete items," which is close to the normal-processing cost for that page.
- ⚠️ Assembly is still redone unconditionally for any item that wasn't already `PUBLISHED` — fine unless assembly itself turns out to be expensive for a given report type.

### Option B — In-process pipeline with a resolve checkpoint

Same as A, but `WorkItem.status = RESOLVED` is persisted mid-flight (with the resolved data or a pointer to it) so recovery can skip resolution for items that individually reached that checkpoint before the crash.

- **[v05] Its edge over Option A has narrowed, not disappeared.** Since recovery already batch-resolves the whole affected page regardless of individual item checkpoints (a page-level query is roughly the same cost whether it's serving 10 already-resolved items or 500 fresh ones), B's per-item `RESOLVED` checkpoint mostly avoids *redundant* resolution within a page that partially succeeded — a smaller saving than v04 implied, where the comparison was against naive per-item resolution.
- ⚠️ **[v05]** Storing resolved hierarchies for potentially thousands of in-flight items per run is a real storage/retention cost, not "negligible" — it needs its own cleanup/expiry job (e.g., purge resolved-data payloads once the owning `WorkItem` reaches a terminal state, on a schedule independent of the main tables' retention).
- ⚠️ Still one deployable, one pipeline — a refinement of A, not a structural change.

### Option C — Staged pipeline, independent sweeper per stage

`Resolver` / `Assembler` / `Publisher` as separate clustered Quartz sweepers, each owning one transition. Item-level recovery is implicit (a stale item is picked up by whichever sweeper owns its current state). The run-level paging-checkpoint recovery sweeper still applies, since paging resumption is a run-level concern independent of how many item-level stages exist.

- ✅ Best operational visibility and independent per-stage scaling.
- ⚠️ Three components to build, deploy, monitor; more Quartz configuration.
- ⚠️ Per-item latency is the sum of sweep intervals unless stages chain directly on completion — which starts to resemble Option A/B with more moving parts.

---

## Comparison **[v05 — updated]**

| | A: In-process, redo-on-recovery | B: In-process, resolve-checkpointed | C: Staged sweepers |
|---|---|---|---|
| Build effort | Lowest | Medium | Highest |
| Recovery cost (post-fix) | Batched page resolve + per-item assembly redo | Batched page resolve (mostly) skipped for already-`RESOLVED` items + per-item assembly redo | Resumes from last stage per item |
| Extra schema/ops cost | None | Resolved-data storage + its own retention job | 3 sweepers + retention/monitoring per stage |
| Operational visibility | Coarse | Coarse | Fine-grained |
| Best fit if... | Batched-page recovery already makes resolution cheap enough — likely true here | Individual resolve is still meaningfully expensive even at page-batch granularity, and you're willing to own resolved-data retention | Volume/observability genuinely demands per-stage scaling, independent of recovery cost questions |

**[v05] Recommendation revised: Option A.**
The batched-page recovery fix removes the strongest argument for B — B's schema was justified by "recovery re-resolves per item, which is expensive," and that's no longer true once recovery batches by page like normal processing does. What's left of B's advantage is real but smaller (skipping redundant resolve within a partially-completed page) and comes with a genuine, previously understated cost (retaining resolved-data payloads for thousands of in-flight items, with its own cleanup job). Given that, **A is the better default**: simplest to build and reason about, and its recovery cost is now close to normal-processing cost for the affected pages. Move to B only if profiling shows resolution — even batched — is still a meaningful fraction of recovery time, which would need to be measured, not assumed. C remains reserved for a concrete, stated per-stage observability/scaling need.

---

## Open items for a next pass

- Sequence diagram for the FAQ Q2 race, worked through against Option A's final schema and recovery design.
- `ScopeClaim` TTL value, chosen relative to expected per-config processing time and the recovery-detection interval (given the newly stated claim-expiry-during-recovery consequence).
- The `ProcessedInboundMessage` retention floor, set from the actual backout-queue/redelivery-limit configuration on `CAMT.ONDEMAND.QUEUE` and `CAMT.PHT.QUEUE`, not estimated.
