# Commander Redesign — Review Log (v01 → v07)

A round-by-round record of how the Commander architecture options evolved across seven
revisions: an external design agent proposed, a reviewer (with knowledge of the prior
implementation's internals) critiqued, the next revision folded the feedback in. Captured here
so the reasoning behind the final shape survives outside the chat transcript.

## Starting point

**`message-pipeline/goal.md`** — a lean brief: purpose (generate CAMT report messages, publish JSON
`ReportMessage` to one IBM MQ queue per report type; a downstream Executor renders the actual
reports), six report types, three inbound triggers (scheduled / on-demand / PHT), fixed
constraints (Spring Boot, IBM MQ, SQL Server, Quartz), multi-pod deployment where any pod can
die mid-run — plus a "processing rules" section (bundling; reporting window derived from the
*scheduled* time; feature flags; on-demand re-requests are not duplicates; all three triggers
publish to MQ; concurrent triggers can overlap on the same scope; data resolution must be
batched per page; full `ReportConfig` pass-through).

**`faq.md`** — supporting Q&A: what "unit of work" / "concurrent triggers overlap" / "batched
resolution" mean (Q1–Q3); there is no caller-supplied on-demand request ID — use
`JMSMessageID` for inbound-redelivery dedup and a Commander-minted execution ID for outbound
identity (Q4); the cross-trigger lock is `(config, window)`-scoped, with the outbox unique
constraint as the real guarantee (Q5).

---

## Round by round

### v01 — three options, differing by duplicate-prevention mechanism

**Proposed:** unit of work = one report per account/config per point in time, tracked
individually (PENDING/DONE). Common design: clustered Quartz, a scheduled-run tracking table,
a periodic stuck-run check; on-demand/PHT lean on queue redelivery. Options: (1)
publish-then-mark-done with a fingerprint (account + report type + date) dedup check; (2)
transactional outbox — write message + mark done atomically, separate relay to MQ; (3) as (2)
with the publisher extracted as a shared component. Recommended (2).

**Review flagged:**
- "Unit of work = one account" breaks **bundling** — bundled config = one message per payment
  type (all-or-nothing), unbundled = one per account, no-scope = one per config. Ledger
  granularity must equal message granularity.
- Fingerprint holes: must be built from the **scheduled** period, not wall-clock; a
  fingerprint-skip **suppresses legitimate on-demand re-requests**; SELECT-then-check is a
  race — needs a DB `UNIQUE` constraint, not a query.
- On-demand and PHT **also publish to MQ** — the double-publish risk applies to them too; the
  outbox/unique key must be the universal publish path.
- Missing (architecture, not detail): cross-trigger isolation; poison rows; recovery
  ownership arbitration; per-config resolution N+1 at volume; a stated Spring Batch position.
- Option 3 is not a distinct architecture — it's Option 2 with good modularity.
- None of the three is literally exactly-once; all are at-least-once + idempotent consumer.

### v02 — re-cast as three options, differing by internal structure

**Changed:** outbox + `UNIQUE` constraint on logical identity moved into a settled
**foundation** (no longer an option axis). Logical identity spelled out per trigger; inbound
`JMSMessageID` dedup separated from outbound dedup; `(config, window)` advisory lock;
batched-by-page resolution; window from scheduled fire time; feature-flag skip semantics.
Options re-framed as **(1) single shared pipeline / (2) hexagonal adapters + core + publisher
/ (3) staged state-machine with per-stage sweepers**. Recommended (2).

**Review flagged:**
- Options 1 and 2 are the same runtime — only code modularity differs. The real fork is
  in-process pipeline vs staged state machine.
- **Recovery is only actually designed for Option 3** — Options 1/2 hand-wave detection,
  ownership arbitration, and where the recovery loop runs.
- Poison items still unaddressed in any option.
- Spring Batch still not mentioned — Option 3 hand-rolls its job-repository model.
- Relay throughput ceiling (single-pod) named for latency but not throughput.
- Work-item row creation isn't a cheap "step 0" — fan-out isn't known without resolving scope.
- PHT outbound identity vaguer than the on-demand answer — risks suppressing a legit re-push.
- Missing the pragmatic middle: Option 2 structure + one checkpoint column + one recovery
  sweeper.

### v03 — corrections folded into the foundation; options become A/B/C

**Changed:** foundation corrections — (1) row creation **interleaved** with resolution, page
by page; (2) PHT gets a minted acceptance ID like on-demand; (3) poison items —
`attempt_count` + terminal `FAILED_POISON` + manual redrive; (4) stated Spring Batch position
(not adopted, with reasons); (5) relay sharding (claim-batch, or partition by `report_type`).
Recovery designed as a shared component: `Run` heartbeat, clustered sweeper, CAS ownership,
resume = re-drive non-terminal items. First concrete **table schemas**. Options recast as
**A (in-process, redo-on-recovery) / B (in-process, resolve-checkpoint) / C (staged
sweepers)**. Recommended **B**.

**Review flagged:**
- **Scheduled-run recovery ignores the paging cursor** — interleaved creation means work
  items for un-reached configs don't exist, so the sweeper can't know to create them. `Run`
  needs a persisted paging checkpoint.
- **On-demand/PHT recovery-sweeper races broker redelivery** → two execution IDs for one
  logical unit → duplicate. Sweeper should be scheduled-only.
- "Outbox row already exists for my key" must be an **explicit success path** (SELECT + mark
  PUBLISHED), or recovery bounces items to `FAILED_POISON`.
- `ScopeClaim` has no crash-release story — a dead claim blocks other triggers forever.
- `logical_key` as a delimited string is fragile — use typed columns / multi-column unique.
- Option B's `ASSEMBLED` checkpoint is redundant with the outbox row existing.

### v04 — recovery scope corrected; schema hardened

**Changed [v04]:** recovery sweeper **scoped to scheduled runs only**, with the
redelivery-race reasoning stated. `Run.last_config_id_processed` paging checkpoint + "resume
paging" as part 2 of recovery. `Run.recovery_attempt_count` → `ABANDONED` give-up path.
`logical_key` replaced with a typed 7-column `UNIQUE` constraint (sentinel `execution_id` for
scheduled). `ScopeClaim.expires_at` TTL + lifecycle (TTL expiry *is* the crash release).
"Outbox row already exists" as a named success branch. `SKIPPED_FLAG_OFF` stated terminal.
Recommended **B**.

**Review flagged:**
- `WorkItem` creation isn't idempotent — re-paging a partially-created page makes duplicate
  rows unless there's a natural-key `UNIQUE` or a stated one-transaction-per-page boundary.
- Executor still gets at-least-once — `ReportMessage` must **carry its own dedup identity
  tuple** (none of which are `ReportConfig` fields).
- Per-item recovery **re-resolves per item** — the exact N+1 the batched design avoids.
  Recovery should re-page in batches; this also shrinks B's advantage over A.
- Minor: orphaned on-demand/PHT `Run` rows leak `IN_PROGRESS`; `ProcessedInboundMessage`
  retention must exceed the redelivery window; B's resolved-data storage cost understated;
  `ScopeClaim` TTL-during-recovery widens the scheduled+on-demand overlap window.

### v05 — idempotency, message contract, batched recovery; recommendation flips to A

**Changed [v05]:** `UQ_WorkItem_Identity` on `(run_id, config_id, scope_key, window_start,
window_end)`; idempotent page inserts; checkpoint advanced in its own small transaction.
`ReportMessage` dedup-identity fields written into the contract explicitly. **Recovery
batch-resolves per page**, not per item — "drain existing items" and "keep paging" become one
batched mechanism. Orphaned non-scheduled `Run` cleanup job. `ProcessedInboundMessage`
retention floor documented as a known residual. **Recommendation flipped from B to A** —
batched recovery removes the cost B's checkpoint was defending against.

**Review flagged:**
- **Recovery re-resolves from current DB state, which can diverge from the original run's
  fan-out** — a config whose scope shrank strands an orphaned `WorkItem` into a false
  `FAILED_POISON`; a scope that grew silently expands the run past "as-of scheduled time."
- `ScopeClaim` acquire as written (`INSERT ... WHERE NOT EXISTS`) throws a PK violation
  against an existing expired row — needs `MERGE` / upsert.
- Nits: `RESOLVED` is Option-B-only; no `CHECK` on `execution_id` for non-scheduled runs;
  ordering never ruled in or out.

### v06 — recovery reconciliation; MERGE-based claim

**Changed [v06]:** `WorkItem` rows are the **authoritative fan-out** — recovery re-drives the
existing set; batch-resolve fetches *data*, not membership. A genuinely-vanished `scope_key` →
new non-alertable terminal state **`OBSOLETE`**. Forward paging past the checkpoint still
resolves fresh. `ScopeClaim` acquire replaced with a `MERGE` upsert. Nits folded: `RESOLVED`
labeled B-only; `CK_Run_ExecutionId_Required` CHECK; explicit no-cross-message-ordering
statement, flagged as an Executor-team contract point.

**Review flagged:**
- The `MERGE` needs `WITH (HOLDLOCK)` on the target — otherwise concurrent first-time claims
  race `WHEN NOT MATCHED` → PK violation or deadlock. This is the normal case for this table.
- `OBSOLETE` needs a "resolution actually succeeded" guard — a transient resolve failure must
  not masquerade as "scope genuinely gone."

### v07 — closes out

**Changed [v07]:** `MERGE ScopeClaim WITH (HOLDLOCK)`, noted as load-bearing. The resolve
step returns a **tri-state** (found / confirmed-absent / query-failed); only *confirmed-absent*
retires an item as `OBSOLETE`, query-failed goes through the ordinary `attempt_count` →
`FAILED_POISON` path.

**Review:** no further architecture or schema issues. Phase closed on **Option A**.

### v08 — post-review product decisions (not a review round)

Three constraints confirmed by the product owner, which **removed** machinery rather than adding it:

- A scheduled run and an on-demand request **may both produce** a message for the same config /
  report type / window, and **both are delivered**, distinguished by trigger metadata.
- **PHT recipients are disjoint, enforced in data:** their `ReportConfig` row has
  `frequency = 'NEVER'`, and the scheduled path selects with
  `report_type = ? AND frequency = ? AND is_active = 1`, so it never picks them up. The PHT
  flow resolves that config by recipient + report type directly.
- Executor accepts two semantically-equal messages as long as trigger metadata distinguishes
  them.

**Change 1 — `ScopeClaim` removed.** The **`ScopeClaim` table and its `MERGE`/`HOLDLOCK`
acquire are gone.** Scheduled and on-demand messages already have distinct outbox identities,
so they never collide on `UQ_Outbox_Identity` and both publish by design — `ScopeClaim` only
ever made one trigger *wait* for the other, contrary to the confirmed intent. `UQ_Outbox_Identity`
alone remains and independently covers the only real double-publish case (a scheduled run vs.
its own crash-recovery / a zombie original pod).

**Change 2 — scheduled `execution_id` is derived, not a sentinel.** Earlier drafts gave every
scheduled-path outbox row an all-zeros `execution_id`; it is now
`uuid5("SCHEDULED|{report_type}|{frequency}|{scheduled_time}")`. The same slot (crash
recovery, a snapped misfire catch-up — both reuse `scheduled_time`) still yields the same id
and dedups; a genuinely distinct firing (a fast-cadence TEST tick, an un-overridden manual
re-run) yields a distinct id and a fresh message. This removes the `END_OF_DAY` fast-TEST
limitation entirely — `END_OF_DAY` can now be fired every few minutes in TEST and produce a
full round-trip on each fire, even though its window ("yesterday 00:00–24:00") never changes.
Production is unaffected (`END_OF_DAY` fires once a day). Consistent with on-demand/PHT, where
`execution_id` already means "which execution produced this."

**Change 3 — `Run` schema fix (from a review of `scheduling/solution_v01.md`).** The scheduling
design leaned on `UNIQUE (report_type, frequency, scheduled_time)` on `Run`, but the `Run` DDL
had no `frequency` column and no such index. Added to `message-pipeline/solution_v08.md`: `frequency
VARCHAR(20) NULL` (with `CK_Run_Frequency_Required` for `SCHEDULED`), `window_start` /
`window_end DATETIME2 NOT NULL` (frozen at creation, so recovery's "keep paging" needs no
recompute and a later window-function change can't alter an in-flight run), and
`UQ_Run_ScheduledSlot` as a **filtered** unique index over `SCHEDULED` rows only — a plain
constraint would wrongly block a legitimate on-demand re-request. Also stated: `Run` creation
hitting that index is a **named success path** (catch, log, exit the job cleanly — no
`JobExecutionException`), mirroring the `Outbox` "already exists" path.

Also proposed (pending confirm): the on-demand path skips-and-logs any supplied config with
`frequency = 'NEVER'` as a cheap guard against a mistaken request. Captured in
`message-pipeline/solution_v08.md`; `message-pipeline/how-it-works.md` and `scheduling/solution_v01.md` updated to match.
Scheduling was also split into its own document (`scheduling/solution_v01.md`) at the product
owner's request.

---

## Decisions that changed during review

| Decision | Early position | Final position | Why it moved |
|---|---|---|---|
| Recommended option | outbox-with-shared-publisher (v01) → hexagonal (v02) → checkpoint-middle **B** (v03–v04) | **A** — in-process, redo-on-recovery | Fixing recovery to batch-resolve per page (v05) removed the per-item re-resolution cost that B's checkpoint existed to avoid |
| What the options differ on | duplicate-prevention mechanism (v01) | pipeline structure: in-process vs staged sweepers (v03+) | outbox + unique constraint became a settled foundation, not a choice |
| Dedup guarantee | fingerprint SELECT-then-check (v01) | `UNIQUE` constraint on a typed identity tuple; the `(config, window)` lock is only an optimization | races between recovery / on-demand / PHT / scheduled make a query-based check unsafe |
| Recovery scope | all triggers swept (v01–v03) | scheduled runs only; on-demand/PHT recover via broker redelivery | sweeping event-driven triggers races redelivery and mints conflicting execution IDs |
| Recovery membership | re-derive fan-out from current data each time | `WorkItem` rows are authoritative; re-resolve fetches data only; vanished scope → `OBSOLETE` | re-derivation diverges from the run's as-of-scheduled-time view — strands or spuriously adds items |

---

## Final shape (v08)

- **Option A**: one in-process pipeline per work item — resolve → assemble → write outbox row
  → mark `PUBLISHED` — shared across scheduled / on-demand / PHT.
- **Correctness backstop**: `UQ_Outbox_Identity` on `(trigger_type, config_id, report_type,
  scope_key, window_start, window_end, execution_id)` — the **sole** guard; there is no
  cross-trigger lock (v08). Delivery to Executor is **at-least-once**; Executor dedupes on the
  same tuple, which `ReportMessage` carries as fields.
- **Cross-trigger independence (v08)**: scheduled and on-demand both publish for the same
  config/window (different `execution_id` → different identity); PHT is held disjoint by a
  `frequency = 'NEVER'` config the scheduled predicate never selects.
- **Unit of work** = one message; `WorkItem` rows are the authoritative fan-out for a run,
  created idempotently, page by page, interleaved with batched resolution.
- **Recovery** (scheduled runs only): heartbeat + clustered sweeper + CAS ownership; re-drive
  existing non-terminal items using freshly-resolved data; resume paging past
  `last_config_id_processed`; `OBSOLETE` for confirmed-absent scope; `ABANDONED` after bounded
  `recovery_attempt_count`.
- **On-demand / PHT recovery**: broker redelivery into a fresh `Run`; inbound dedup on
  `JMSMessageID`; minted execution / acceptance ID in the outbound identity.
- **Terminal `WorkItem` states**: `PUBLISHED`, `SKIPPED_FLAG_OFF`, `FAILED_POISON` (alertable),
  `OBSOLETE` (not alertable).
- **Not adopted**: Spring Batch (step-level restart granularity, a second metadata schema,
  event-driven triggers don't fit the batch-launch model). Option C (staged sweepers) reserved
  for a concrete per-stage scaling / visibility need.

---

## Still open — detailed-design pass, not architecture

- Concrete `ProcessedInboundMessage` retention floor — read from the actual
  backout / max-redelivery configuration on `CAMT.ONDEMAND.QUEUE` and `CAMT.PHT.QUEUE`.
- Confirm the no-cross-message-ordering contract point, and the "two semantically-equal
  messages distinguished by trigger metadata" acceptance, with the Executor team.
- Confirm or drop the on-demand `frequency = 'NEVER'` skip-and-log guard (v08).
- Scheduling design — **`scheduling/solution_v01.md`** (single consolidated doc; the earlier
  draft, the clean brief, the external agent's options pass, and the bikili review were all
  folded in and then removed). Design: one logical schedule per `(report_type, frequency)`;
  one `ReportSchedulingJob` class with 14 data-driven `JobDetail`s (no per-report-type
  subclasses); crons generated from a declarative spec; boundary frequencies on a single cron
  with the sequence derived by snapping the fire time; pure window function (rolling /
  boundary / calendar-day) with an explicit DST rule; `UNIQUE (report_type, frequency,
  scheduled_time)` on `Run`; `requestRecovery(false)` (the pipeline's sweeper owns recovery);
  concurrent adjacent-window runs are intended (no `@DisallowConcurrentExecution`); manual
  runs via `scheduler.triggerJob` + a `scheduledTimeOverride`. Open within it: misfire policy
  (needs a real Quartz test), `EVERY_2/4_HOURS` window shape (rolling-matches-production vs
  boundary), the eight `EIGHT_TIMES_PER_DAY` times, DST confirmation, pause/resume in v1.

> The `faq.md` Q2 race (scheduled + on-demand on the same `(config, window)`) and the
> `ScopeClaim` TTL are **no longer open items** — v08 removed the claim; that case is now just
> "both publish, distinguished by trigger metadata."
