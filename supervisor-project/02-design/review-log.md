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
  concurrent adjacent-window runs are intended (no `@DisallowConcurrentExecution`).
  **Decisions folded in since:**
  - Misfire policy = **do nothing**. No automatic catch-up; missed slots recovered only by an
    explicit backfill.
  - `EVERY_2/4_HOURS` = **boundary**, first window anchored to `00:00` — a deliberate
    divergence from the legacy system's rolling behaviour (**parity note for cutover**).
  - `EIGHT_TIMES_PER_DAY` boundaries = `03:00, 06:00, 08:00, 10:00, 12:00, 15:00, 18:00, 21:00`
    (config-revisable later).
  - **Cron is generated** from the declarative spec (raw `cron-override` for TEST only) —
    the hand-written-cron question is closed.
  - **Admin endpoints in v1:** `run`, `backfill` (explicit slot list or `from`/`to` range →
    one job per slot), `pause`, `resume` (forward-only), `status`.
  - **DST rules chosen:** spring-gap boundary → shift forward; fall-back boundary → earlier
    occurrence; transition-day windows are calendar windows (elapsed length varies by ±1 h).
  - **`Run` schema:** `frequency` + `window_start`/`window_end` columns and the filtered
    `UQ_Run_ScheduledSlot` index added to `solution_v08.md`.
  - **Business timezone:** `Europe/Stockholm`.

  **Closed since:** the business confirmed both the DST resolution rules and the
  `EIGHT_TIMES_PER_DAY` fire times / window rule (00:00→03:00, 03:00→06:00, … as designed) — no
  change to either. Only a build-time Quartz test remains (`solution_v01.md` §16).

  **`requestRecovery` flipped from `false` to `true` (reader feedback on `solution.md` §8:
  "are you confident there will be no crash between a pod picking up a firing and creating its
  `Run` row? shouldn't we use `requestRecovery(true)`?").** The original `false` choice assumed
  Quartz's own trigger-level recovery would *race* the pipeline's heartbeat sweeper. On
  inspection that assumption doesn't hold: the scheduling job's own logic never attempts to
  resume in-progress work — it only ever creates a `Run` or, if one already exists, exits via
  the same `UQ_Run_ScheduledSlot` no-op used for any duplicate firing. So a Quartz-recovered
  re-fire and the sweeper cover **disjoint** failure windows (before vs. after `Run` creation)
  rather than competing, and `requestRecovery(false)` was leaving a real gap: a pod dying in
  the moment between picking up a firing and committing its `Run` row was silently lost —
  neither the misfire policy (nothing became due) nor the sweeper (no `Run` row to find) could
  see it. Flipped to `true`; `solution_v01.md` §5/§9/§10, `solution.md` §4/§8, and
  `how-it-works.md` §7 updated; a matching Quartz test added to `solution_v01.md` §16.

  **Misfire policy generalised to cover a deliberate pause, not just a crash** (reader
  feedback on `how-it-works.md` §8: does an operator pausing a trigger for a production issue
  also cause missed firings?). Answer: yes, and it was already handled by the same
  `MISFIRE_INSTRUCTION_DO_NOTHING` mechanism — a paused trigger's fire times that fall due
  while it's paused are in the past by the time it's resumed, so Quartz applies the same
  do-nothing instruction to them. §9 now states this explicitly as a second cause alongside
  "cluster was down", and §8/how-it-works.md's missed-firing section documents the intended
  operator workflow: pause → fix → resume → **explicit** backfill of whichever slots the
  business agrees need recovering. No design change — a documentation gap, now closed.

  **`scheduling/how-it-works.md` retired.** Once `scheduling/solution.md` existed as a single
  self-contained doc covering the same ground in the same accessible register,
  `how-it-works.md` was redundant — removed. `solution_v01.md` remains, as the one place that
  still carries implementation detail `solution.md` deliberately leaves out (the full
  generated-cron table, startup validation rules, the properties/edge-case appendix). No other
  doc pointed at `how-it-works.md` by path; the mentions above are historical.

  **`scheduling/solution_v01.md` renamed to `scheduling/implementation-reference.md`** (reader
  feedback: the original goal was one document, and keeping a same-generation `solution_v01.md`
  alongside `solution.md` read as an unfinished merge rather than a deliberate split). Content
  unchanged; only the framing and filename changed — it's now explicitly positioned as
  `solution.md`'s implementation-detail companion, not a parallel design doc, with a pointer
  each way between the two files. The four cross-references from `message-pipeline/solution_v08.md`
  and `message-pipeline/how-it-works.md` were repointed to `../scheduling/solution.md`.

> The `faq.md` Q2 race (scheduled + on-demand on the same `(config, window)`) and the
> `ScopeClaim` TTL are **no longer open items** — v08 removed the claim; that case is now just
> "both publish, distinguished by trigger metadata."

---

## Data-retrieval — separate track

The "resolve" step in `message-pipeline/how-it-works.md` (§4, *Scheduled*, step 3) — turning a
set of `ReportConfig`s into fully-resolved input for message assembly — is its own concern
with its own decisions (staged reads vs. join; SQL Server IN-list/parameter limits; keyset
pagination; batched recipient lookup; pure row→tree assembly; the shared core across
scheduled/on-demand/PHT; the tri-state resolve result recovery needs). Split into
`02-design/data-retrieval/` (sibling of `message-pipeline/` and `scheduling/`).

- `data-retrieval/goal.md` — clean-slate brief.
- `data-retrieval/solution_v01.md` — external agent's 3-option pass (per-level batch queries /
  stored proc multi-resultset / flattened join); recommends per-level batch queries.
- `data-retrieval/solution_v02.md` — **the design**. Commits to **staged per-level batch reads
  + a pure in-memory assembler** (the shape the legacy `bikili` code runs, hardened):
  6 fixed round trips per page (keyset config page → scopes via plain `IN` → payment types /
  accounts / aliases via a `dbo.BigIntIdList` TVP → recipients via plain `IN`), then a
  DB-free assembler. Two modes on one core: `resolvePage` (fresh) and `reResolve` (recovery) —
  where **`reResolve` = `resolvePage` + a per-`WorkItem` presence check** yielding the
  `FOUND / CONFIRMED_ABSENT / QUERY_FAILED` tri-state, so there is no second query
  implementation. Output is a **typed `ResolvedConfig` record** (not an opaque map), carrying
  both surrogate `id` and business `configId`. PHT balance merge is a pure post-resolution
  function. One schema addition: the TVP type (chunked-`IN` is the fallback). Proposes a
  concrete `scope_key` grammar to sign off jointly with the pipeline.

  Open: `scope_key` grammar sign-off; TVP schema-addition approval; page-size tuning;
  whether `reResolve` needs recipient data.

- **Diagrams (`.drawio`, mono).** `data-retrieval/data-fetch-flow.drawio` (six staged reads /
  dependency view / recovery tri-state), `scheduling/scheduling-flow.drawio` (trigger sources
  / scheduled job lifecycle / window shapes / clustered Quartz), and
  `message-pipeline/pipeline-flow.drawio` (end-to-end / outbox + delivery / recovery /
  `WorkItem` state machine). Visualisation only — no design change; each solution doc links its
  file near the top.

- **Efficiency pass on `solution_v02.md`** (in-conversation). Added: the required-indexes
  table (headline: composite `ReportConfig (ReportType, ReportFrequency, IsActive, Id)`);
  the `OPTION (RECOMPILE)` note on the TVP stages; the *Bounding page width* guards
  (`max-accounts-per-config`, `page-row-hard-limit`). Expanded §11 "Why not the alternatives"
  with per-config-anti-pattern round-trip maths and the "when would this win" line for
  Options B and C.

- **External review of `solution_v02.md` against `goal.md`** (folded in). Three points:
  (1) recovery's Level-1 fetch-by-surrogate-`Id` had no defined query — §2 now states it
  explicitly as a third Level-1 variant (`Id IN (:ids)`, **no `IsActive` filter**).
  (2) the tri-state didn't cover a config that went inactive since the original run — §2's
  `reResolve` now branches per config: config deleted **or** `IsActive = 0` →
  `CONFIRMED_ABSENT` for every ref under it, with a distinct log reason.
  (3) the assembler's account/alias mutual-exclusivity check said "throw" while the sibling
  account-ceiling guard "isolates" — §5 now makes the violation a per-config poison item
  (isolated, page continues), consistent with the ceiling guard, plus a note that a high
  violation *rate* signals a staged-read bug not bad data. Also softened the
  `OPTION (RECOMPILE)` claim: it fixes row count, not value-distribution stats — fine for the
  straight-join shapes here, revisit if a future TVP query adds a second selective predicate.
