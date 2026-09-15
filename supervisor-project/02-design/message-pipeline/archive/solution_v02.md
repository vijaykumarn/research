Good — this closes the remaining gaps. Here's the redesign, incorporating everything: the outbox/unique-constraint pattern, JMSMessageID-based inbound dedup, execution-ID-based outbound dedup for on-demand, (config, window)-scoped advisory locking with NACK-and-retry on contention, Quartz clustering, and batched data resolution.

## Foundation shared by all options

These aren't architecture choices — they're now settled requirements, so every option below includes them:

- **Quartz, clustered mode, JDBC JobStore in SQL Server** — guarantees exactly one pod fires a given scheduled trigger.
- **Unit of work = one message.** A scheduled/on-demand/PHT trigger expands into N units of work (bundled → one per payment type; unbundled → one per account/alias; no-scope → one config-only). Granularity is decided once, up front, per config.
- **Outbound dedup = unique constraint, not a lock.** A `PublishedMessage` (or "sent ledger") table has a `UNIQUE` constraint on the message's logical identity. Two racing attempts to produce the same message both try to insert; one wins, one gets a constraint violation and simply discards its work. This is the actual correctness guarantee.
- **Logical identity, per trigger type:**
  - Scheduled: `(configId, reportType, paymentType/account-or-none, window)` — stable, content-only.
  - On-demand: the same content key **+ execution ID** (a UUID minted when Commander accepts the on-demand message off the queue), so re-submissions are never suppressed.
  - PHT: content key + the external engagement/push identifier, since PHT is inherently "produce this one now."
- **Inbound dedup (separate from outbound) = JMSMessageID ledger.** Before processing an on-demand/PHT message, check a small `ProcessedInboundMessage` table keyed by JMSMessageID. If present, it's a broker redelivery of already-completed work — ack and drop. If absent, process, and record the JMSMessageID as part of the same transaction/outbox write that records the outbound message(s).
- **Cross-trigger claim = advisory lock on (configId, window), acquired per config, released once that config's message(s) for that window are durably written.** Not a correctness mechanism — just cuts down on wasted work. On contention (can't get the claim), on-demand/PHT NACK for redelivery with backoff; scheduled runs skip and let their own recovery sweep retry later.
- **Data resolution is batched by page of config IDs** (e.g., 500 at a time) at the DAO layer — one query per level of the hierarchy per page, not per config. This is a data-access-layer discipline, not an architectural fork, so it applies identically in every option.
- **Window computed from the trigger's scheduled fire time**, not wall-clock — stored on the run/work-item record so a resumed run recomputes nothing.
- **Feature flags checked per report type / per config before build and before publish** — a false flag is a logged skip, marked as such in the tracking row (not left PENDING forever, not counted as failed).
- **Full config pass-through** happens in the message-assembly step, mapping every `ReportConfig` field into `ReportMessage` regardless of Commander's own logic — a pure data-carrying concern, doesn't affect any option below.

Where the options genuinely differ is **how the pipeline is internally structured and where the outbox/publish step lives.**

---

## Option 1 — Single shared pipeline, synchronous outbox-then-relay

**Structure:** One internal `ReportPipelineService` that all three triggers call directly (Quartz job, JMS `@JmsListener` for on-demand, JMS `@JmsListener` for PHT). It does: resolve → assemble → **write to outbox table** (message body + logical identity, unique constraint enforced) in the same local transaction as marking the work item done / recording the JMSMessageID. A separate lightweight **relay** — a Quartz job on a short fixed interval, or a `@Scheduled` poller guarded by `sp_getapplock` so only one pod runs it at a time — reads unsent outbox rows and pushes them to MQ, marking them sent.

```
Scheduled trigger ─┐
On-demand listener ─┼─▶ ReportPipelineService ─▶ Outbox table ─▶ Relay poller ─▶ MQ
PHT listener ───────┘        (resolve/assemble/lock/dedup)
```

**Trade-offs**
- ✅ Simplest mental model — one pipeline, one outbox, one relay. Easiest to build and operate first.
- ✅ The risky "did I publish yet" moment is fully isolated to the relay, which only does one thing.
- ⚠️ Relay introduces a small, bounded publish latency (typically seconds) between "message ready" and "message on MQ" — fine here since there's no fixed SLA, but worth naming.
- ⚠️ As report types/volume grow, the pipeline service can become a large class with a lot of branching (bundling logic, pagination-free but still per-scope fan-out, three trigger-specific pre-steps). Manageable at current scope (6 report types), but will need internal sub-packaging discipline over time.

---

## Option 2 — Ports-and-adapters (hexagonal), trigger adapters + shared core + independent publisher

**Structure:** Same overall flow, but explicitly modularized: three thin **adapters** (Quartz job, on-demand listener, PHT listener) each translate their trigger into a common internal command (`ProduceReportCommand` with scope + window + trigger metadata), and hand off to a **core domain module** that owns resolution, bundling/fan-out, assembly, locking, and dedup — with zero knowledge of Quartz or JMS. The core writes to the outbox; a **Publisher module** (its own package, its own poller, its own MQ session handling) is the only thing that touches the MQ `JmsTemplate`.

```
Quartz adapter ────┐
OnDemand adapter ──┼─▶ [Core: resolve → bundle/fan-out → assemble → claim+dedup] ─▶ Outbox ─▶ Publisher module ─▶ MQ
PHT adapter ────────┘
```

**Trade-offs**
- ✅ Trigger-specific quirks (PHT's fixed-width parsing, on-demand's execution-ID minting, scheduled's paging) stay in their adapters and never leak into the core logic — the core only ever sees "produce this scope for this window."
- ✅ Easiest to unit-test: core pipeline tested with no JMS/Quartz/MQ at all; adapters tested for translation only; publisher tested for MQ semantics only.
- ✅ If a fourth trigger ever appears (e.g., a REST-triggered path later), it's a new thin adapter, not a change to the core.
- ⚠️ More upfront structuring (module boundaries, an internal command object, explicit interfaces) than Option 1 for the same runtime behavior — pure design/discipline cost, not a different runtime architecture.
- ⚠️ Slightly more indirection to trace through when debugging end-to-end, though each piece is individually simpler.

---

## Option 3 — Staged pipeline with persisted state machine per work item

**Structure:** Each unit of work is a row that moves through explicit persisted states: `CLAIMED → RESOLVED → ASSEMBLED → PUBLISHED` (or `SKIPPED_FLAG_OFF` / `FAILED`). Rather than one method call doing resolve-assemble-publish in sequence in-process, each stage is its own small Quartz-driven sweeper that picks up rows in the prior state, does its stage, and advances the row — still within the same service/deployment, just structurally decomposed by stage rather than by trigger.

```
Triggers create CLAIMED rows (with lock+dedup already applied)
   │
   ▼
[Resolver sweeper]  CLAIMED    → RESOLVED   (batched per page, as in Q3)
   ▼
[Assembler sweeper] RESOLVED   → ASSEMBLED  (bundling/fan-out, full pass-through)
   ▼
[Publisher sweeper] ASSEMBLED  → PUBLISHED  (outbox write + MQ send, unique constraint)
```

**Trade-offs**
- ✅ Best resumption story of the three: a crash mid-run leaves rows sitting in whatever state they reached — no in-flight in-memory state to lose, and any pod's next sweep continues exactly where it left off, stage by stage.
- ✅ Strong observability for free: "how many reports are stuck in RESOLVED right now" is a single query — useful given the thousands-of-configs scale.
- ✅ Each stage can be scaled/tuned independently later (e.g., resolution is DB-heavy, publishing is MQ-heavy) without touching the others.
- ⚠️ Real added complexity: more tables/columns, more sweepers to configure in Quartz, more places for a bug in state transition to strand a row. Only pays for itself if the volume/observability need justifies it.
- ⚠️ End-to-end latency per report is naturally a bit higher (waits for the next sweep interval at each stage) unless stages are chained to trigger each other immediately on success, which reintroduces some of Option 1/2's synchronous feel anyway.

---

## Comparison at a glance

| | Option 1: Shared pipeline | Option 2: Hexagonal | Option 3: Staged state machine |
|---|---|---|---|
| Build effort now | Lowest | Medium | Highest |
| Resumption granularity | Per work item (outbox row) | Per work item (outbox row) | Per work item, per *stage* |
| Testability | Moderate | High | High |
| Best fit if... | You want this shipped and correct soonest | You expect more trigger types or want strict separation of concerns | Volume keeps growing and per-stage visibility/scaling matters |
| Publish latency | One relay hop | One relay hop | One or more sweep hops |

**My recommendation:** start with **Option 2**. It costs little more than Option 1 to build, and given that Commander already has three quite different trigger sources with real differences in their dedup/lock needs (as your Q1/Q2 answers show), keeping that variability contained in adapters — rather than as branching inside one pipeline class — will pay off quickly. Option 3 is worth revisiting later if the sweeper-per-stage observability becomes something you actually need operationally, but I wouldn't start there.

Want me to go one level deeper on any of these — e.g., the actual table schemas (outbox, claim/lock table, processed-inbound-IDs), or a sequence diagram for the "scheduled run overlapping with on-demand" race from your FAQ Q2?