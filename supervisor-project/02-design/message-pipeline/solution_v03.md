You're right on every point — let me fix this properly rather than patch around it. I'll fold the corrections into the foundation, then re-present as the real fork: **Option A (in-process pipeline) vs Option B (checkpoint-column middle) vs Option C (staged state machine)**, each with recovery actually designed, not hand-waved.

## Foundation — corrections

**1. Unit-of-work creation is interleaved with resolution, not a step 0.**
For a scheduled run: page through `ReportConfig` (e.g., 500 at a time) → batch-resolve that page's scope/payment-type/account hierarchy → *now* the fan-out is known → insert the work-item rows for that page (bundled: one row per payment type; unbundled: one row per account/alias; no-scope: one row) → move to the next page. So "create rows" and "resolve" are the same loop, one page at a time, not two sequential phases. This also bounds memory — you never hold all thousands of rows' resolved data at once, just one page's worth.

**2. PHT gets a minted acceptance ID, same treatment as on-demand.**
You're right that `messageDate + messageTime + accountOwner` isn't a safe identity — a deliberate re-push landing in the same slot would be wrongly suppressed. Fix: when the PHT listener accepts a message off `CAMT.PHT.QUEUE`, mint a Commander-side **PHT acceptance ID** (UUID) the same way on-demand mints an execution ID, and fold it into that message's outbound logical identity. Inbound redelivery dedup still uses `JMSMessageID` (that part was fine); this fix is specifically about the outbound key.

**3. Poison work items — attempt count and a real terminal state.**
Work-item tracking gets an `attempt_count` and `last_error` column. A configurable max (e.g., 5) moves a row to `FAILED_POISON` instead of retrying forever. `FAILED_POISON` is a genuinely terminal, alertable state — it should page/notify, and support a manual redrive (an operator action that resets it to `PENDING` with `attempt_count = 0` after the underlying data issue is fixed). This is schema-level, not an afterthought, so it's in the table design below.

**4. Spring Batch — a stated position, not an omission.**
Spring Batch's `Job`/`Step`/`ItemReader-Processor-Writer` model with a job repository gives you restartability and chunked processing out of the box — exactly the shape Option C reinvents. I'd still lean toward **not** adopting it here, for stated reasons:
- Spring Batch's restart model resumes a **step**, not an arbitrary work item mid-page — it's built around chunk-oriented reads, less natural for "resume this specific config's unbundled fan-out that got half-published."
- It adds its own metadata schema (`BATCH_JOB_INSTANCE`, `BATCH_STEP_EXECUTION`, etc.) alongside the outbox/claim/tracking tables you already need — two persistence models to reason about for one concern (resumability).
- Commander's three trigger types (scheduled/on-demand/PHT) don't map cleanly onto Spring Batch's single-job-instance-per-launch model; on-demand and PHT are event-driven, not batch-launched.
- If Commander's needs grow toward genuinely large batch windows with complex chunk/skip/retry policies beyond what's described here, Spring Batch becomes worth revisiting — it's a reasonable *later* option, not a wrong one, just not the best fit for this specific per-message-identity resumability need now.

**5. Relay throughput — name the ceiling, name the fix.**
A single `sp_getapplock`-guarded relay pod is fine for latency but becomes a bottleneck at thousands of outbox rows per run pushed one at a time. Fix: shard the relay's claim — multiple pods can run the relay concurrently, each claiming a batch of unsent outbox rows via `UPDATE ... SET claimed_by = @pod, claimed_at = SYSUTCDATETIME() WHERE status = 'PENDING' AND (claimed_by IS NULL OR claimed_at < stale_threshold)` with `READPAST` locking hints, or simpler: partition by `report_type` (6 types → up to 6 relay workers, one per queue, naturally parallel since each writes to a different MQ queue anyway). No `sp_getapplock` singleton needed once claiming is row-level.

---

## Recovery — designed properly, as a shared component across all options

This is the same design in every option below; only *what* it resumes differs.

- **Detection:** every run (scheduled/on-demand/PHT) has a `run` record with `started_at` and `heartbeat_at`, updated periodically by the owning pod while active. A **recovery sweeper** — itself a Quartz job, clustered, firing every N minutes — queries for runs where `heartbeat_at` is older than a stale threshold (e.g., 3× the expected heartbeat interval) and `status != COMPLETED`.
- **Ownership arbitration on recovery:** the sweeper claims a stale run with a conditional update — `UPDATE Run SET owner_pod = @me, heartbeat_at = SYSUTCDATETIME() WHERE run_id = @id AND heartbeat_at = @lastSeenValue` (optimistic, compare-and-swap style). Only one of several simultaneously-restarting pods wins the claim; losers move to the next stale run. This solves "several pods restart together and all see the same stuck run."
- **What resuming means:** query that run's work items where `status NOT IN (COMPLETED, SKIPPED_FLAG_OFF, FAILED_POISON)`, and re-drive them from wherever they are — this is why the per-item status column below matters regardless of which option you pick.
- **Where it runs:** a small dedicated Quartz job, clustered like the rest — not bolted onto the main trigger paths, so it keeps running even if, say, the scheduled-trigger code path has a bug.
- **"Once a day report types can't just wait for the next trigger"** — correct, this sweeper is what closes that gap; it's independent of report frequency.

---

## Table schemas

```sql
-- One row per run (scheduled fire, on-demand accept, PHT accept)
CREATE TABLE Run (
    run_id           UNIQUEIDENTIFIER PRIMARY KEY,
    trigger_type     VARCHAR(20) NOT NULL,      -- SCHEDULED | ONDEMAND | PHT
    report_type      VARCHAR(20) NOT NULL,
    scheduled_time   DATETIME2 NOT NULL,        -- window derives from this, never wall-clock
    execution_id     UNIQUEIDENTIFIER NULL,     -- on-demand execution ID / PHT acceptance ID
    status           VARCHAR(20) NOT NULL,      -- IN_PROGRESS | COMPLETED | ABANDONED
    owner_pod        VARCHAR(100) NOT NULL,
    started_at       DATETIME2 NOT NULL,
    heartbeat_at     DATETIME2 NOT NULL
);

-- One row per unit of work (= one prospective message)
CREATE TABLE WorkItem (
    work_item_id     UNIQUEIDENTIFIER PRIMARY KEY,
    run_id           UNIQUEIDENTIFIER NOT NULL REFERENCES Run(run_id),
    config_id        BIGINT NOT NULL,
    scope_key        VARCHAR(200) NOT NULL,     -- payment type / account-or-none, per bundling rule
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    status           VARCHAR(20) NOT NULL,      -- PENDING | RESOLVED | ASSEMBLED | PUBLISHED
                                                 -- | SKIPPED_FLAG_OFF | FAILED_POISON
    attempt_count    INT NOT NULL DEFAULT 0,
    last_error       NVARCHAR(MAX) NULL,
    updated_at       DATETIME2 NOT NULL
);

-- The actual correctness guarantee
CREATE TABLE Outbox (
    outbox_id        UNIQUEIDENTIFIER PRIMARY KEY,
    logical_key      VARCHAR(400) NOT NULL,     -- see identity rules below
    report_type      VARCHAR(20) NOT NULL,
    payload          NVARCHAR(MAX) NOT NULL,    -- serialized ReportMessage
    status           VARCHAR(20) NOT NULL,      -- PENDING | SENT
    claimed_by       VARCHAR(100) NULL,
    claimed_at       DATETIME2 NULL,
    created_at       DATETIME2 NOT NULL,
    CONSTRAINT UQ_Outbox_LogicalKey UNIQUE (logical_key)
);

-- Advisory claim, not correctness-critical
CREATE TABLE ScopeClaim (
    config_id        BIGINT NOT NULL,
    window_start     DATETIME2 NOT NULL,
    window_end       DATETIME2 NOT NULL,
    held_by_run_id   UNIQUEIDENTIFIER NOT NULL,
    acquired_at      DATETIME2 NOT NULL,
    PRIMARY KEY (config_id, window_start, window_end)
);

-- Inbound redelivery dedup (on-demand + PHT)
CREATE TABLE ProcessedInboundMessage (
    jms_message_id   VARCHAR(200) PRIMARY KEY,
    processed_at     DATETIME2 NOT NULL          -- retention-swept periodically
);
```

`logical_key` construction, per trigger (this is the actual dedup guarantee — everything else is optimization):
- Scheduled: `SCHEDULED|{configId}|{reportType}|{scopeKey}|{windowStart}|{windowEnd}`
- On-demand: `ONDEMAND|{configId}|{reportType}|{scopeKey}|{windowStart}|{windowEnd}|{executionId}`
- PHT: `PHT|{configId}|{reportType}|{scopeKey}|{windowStart}|{windowEnd}|{phtAcceptanceId}`

---

## The real fork: three options

### Option A — In-process pipeline (hexagonal), single checkpoint status per work item

Each trigger's adapter creates `Run` + `WorkItem` rows page-by-page as it resolves (per the interleaving fix above). A single core pipeline then, **per work item, in one pod, in-process**: resolve → assemble → write outbox row → advance `WorkItem.status` to `PUBLISHED`. No inter-stage persistence beyond the one status column. The **recovery sweeper** (shared design above) re-picks any non-terminal `WorkItem` and re-runs it from scratch (idempotent because of the outbox unique constraint) — it doesn't resume "mid-assembly," it just redoes the whole item.

- ✅ Simplest runtime: one code path does resolve→assemble→publish per item, easy to trace end to end.
- ✅ Lowest latency: no waiting for a sweep interval between stages.
- ⚠️ Recovery granularity is "redo the whole work item," not "resume from where it got to." Fine since a work item is small (one message), but means a crash right before assembly finishes still costs a full re-resolve for that item.
- ⚠️ If assembly itself is expensive for some report types, redoing it wholesale on every recovery is wasted work — acceptable at current scope, worth watching if that changes.

### Option B — In-process pipeline with checkpoint columns (the pragmatic middle)

Same as Option A, but `WorkItem.status` is actually used as a checkpoint the pipeline **writes and reads mid-flight**, not just a final marker: resolve → persist `status = RESOLVED` (+ the resolved data, or a pointer to it) → assemble → persist `status = ASSEMBLED` (+ payload) → publish → persist `status = PUBLISHED`. The recovery sweeper resumes from whichever checkpoint a stale item last reached, skipping already-done stages, instead of redoing the item wholesale.

- ✅ Most of Option C's resumability (no wasted re-resolution or re-assembly on recovery) without separate sweeper components per stage or inter-stage latency during the happy path — the happy path still runs all stages in one pass, in-process.
- ✅ Requires storing intermediate state (resolved data / assembled payload) somewhere between stages — either in `WorkItem` itself (extra nullable columns) or a side table — small added schema cost.
- ⚠️ Slightly more write volume than Option A (one persist per stage instead of one at the end) — negligible at this scale, worth noting only because it's a real trade-off, not a free win.
- ⚠️ Still one deployable, one pipeline — this is a resumability refinement, not a structural change from A.

### Option C — Staged pipeline, independent sweeper per stage

As before: `Resolver`, `Assembler`, `Publisher` are separate Quartz-driven sweepers, each owning one state transition, running continuously and independently. Recovery is implicit — a stale item just gets picked up by whichever sweeper owns its current state on the next tick; no separate recovery-sweeper logic needed for work-item-level resumption (the shared `Run`-level heartbeat/lease design above still applies for detecting an abandoned *run*, e.g. to know when to mark it `COMPLETED` vs `ABANDONED`).

- ✅ Best operational visibility ("how many items stuck in ASSEMBLED") and independent scaling per stage, as before.
- ✅ Recovery is the most natural of the three — it's just "the sweepers keep running," not a distinct recovery code path.
- ⚠️ Real added complexity: three sweeper components to build, deploy, and monitor instead of one pipeline; more Quartz job configuration.
- ⚠️ Per-item latency is the sum of sweep intervals across three stages unless stages trigger each other directly on completion — which, if you do that, makes it behave like Option B with extra ceremony.

---

## Comparison

| | A: In-process, redo-on-recovery | B: In-process, checkpointed | C: Staged sweepers |
|---|---|---|---|
| Build effort | Lowest | Medium | Highest |
| Recovery cost | Redoes whole item | Resumes from last checkpoint | Resumes from last checkpoint |
| Extra schema vs. shared foundation | None | Intermediate-state columns/table | Same, plus per-stage sweeper config |
| Operational visibility | Coarse (item status only) | Coarse (item status only) | Fine-grained (per stage) |
| Components to run | 1 pipeline + 1 recovery sweeper | 1 pipeline + 1 recovery sweeper | 3 stage sweepers (recovery mostly implicit) |
| Best fit if... | Resolve/assemble are cheap; redo-on-crash is a non-issue | You want real resumability without extra moving parts | Volume/observability genuinely demands per-stage scaling and monitoring |

**My recommendation: Option B.** It gets you the resumability guarantee the brief explicitly asks for (continuing, not just safely redoing, interrupted work) without taking on three independently-deployed-and-monitored sweeper components. Option A is worth it only if you're confident resolve+assemble are cheap enough that "just redo it" is truly a non-issue. Option C is the right call only if per-stage operational visibility is a concrete, stated need — not a hedge against future scale you haven't hit yet.
