# 1. Overview

Corporate Reporting — the product line that delivers CAMT reports to corporate and commercial banking customers — is powered by two purpose-built applications, each owning a distinct, non-overlapping stage of the process:

* **Commander** owns the decision of *when* a report is due. It schedules and validates report generation, resolves the applicable report configuration for each customer, and publishes a report generation request — a structured message, not a report — onto a queue for the Executor application to process. Commander never renders or generates the report itself.

* **Executor** owns the *production* of the report. It consumes report generation requests from the queue and generates the final ISO 20022 CAMT report for delivery to the customer.

Together Commander & Executor gives corporate customers automated, reliable delivery of ISO 20022 CAMT reports — on a schedule, on demand, or reactively when an account event occurs — without losing or duplicating a report even when the underlying infrastructure fails mid-task. Also allowing operations to pause, resume, backfill, or trigger a report on demand directly, without requiring a code change or redeploy.

This solution document is scoped to **Commander** only.

## 1.1 Conceptual Overview

Commander is the CAMT report-generation orchestration component for Corporate Reporting. Its purpose is to produce report-generation requests for CAMT reports — CAMT.052 intraday account reports, CAMT.053 end-of-day statements, and CAMT.054 debit/credit notifications — for every configured recipient, whether triggered by a schedule, an on-demand customer request, or an inbound account event. 

For each report-generation request, Commander determines what is due, retrieves the required data, assembles the request, and reliably publishes it to a queue. Commander's responsibility ends once the request has been durably published.

Commander is made up of three sub-components. 

- **Scheduling** - Reads and validates the deploy-time schedule configuration, builds and registers the underlying Quartz jobs and triggers, keeps the registration clean across redeploys, and runs the clustered scheduler. It also provides the window-calculation logic — rolling, boundary, or calendar-day — as a shared capability.

- **Assembly** - Owns the entry point for every report trigger: a scheduled firing initiated by Scheduling, an on-demand request, or an inbound account balance push. Regardless of the trigger, Assembly follows the same core flow: resolve the reporting window, create a tracking record, verify that the report type is enabled, retrieve the required data, determine the request shape (bundled, unbundled, or configuration-only), build the report-generation request(s), and write each completed request to the outbox. The outbox is the handoff point from Assembly to Delivery.

- **Delivery** - Runs continuously on every pod and drains completed requests from the outbox, publishing them to the outbound queue. It handles publishing failures, retries, and dead-letter recovery.

## 1.2 Scope and Acceptance Criteria

## 1.3 Information Relevant for Prioritization

## 1.4 Architectural Decisions

## 1.5 Assumptions and Pre-Requisites

## 1.6 Logical View - Architecture Scope

Commander's architecture is described here as a set of logical components, grouped under the three sub-components introduced in 1.1, followed by how those components interact.

```mermaid
flowchart LR
    CS(["Scheduling"]) -- fires trigger --> AS[["Assembly"]]
    OD[/On-demand queue/] -- request --> AS
    PH[/Inbound balance push/] -- event --> AS
    AS -- writes to outbox --> DL(["Delivery"])
    DL -- publishes --> Q[(Outbound queue)]
    Q --> EX[Executor]
```

### Scheduling

- **Trigger loader / generator** — reads the deploy-time schedule configuration, expands any entry that lists several report types into one independent schedule per report type, generates the underlying cron timing expression(s) from each schedule's interval or boundary-list specification, and builds and registers the resulting job and trigger definitions with the clustered scheduler at startup.
- **Startup reconciliation** — also at startup, before the scheduler begins running, removes any triggers left over from a prior configuration so nothing fires against a stale schedule.
- **Window function** — a pure calculation that resolves a scheduled fire time to a (window start, window end) pair, exposed as a shared capability Assembly calls into. Every report frequency falls into one of three window shapes:
  - **Rolling** — a fixed look-back ending at the fire time (e.g. the last 30 minutes).
  - **Boundary** — the stretch since the previous checkpoint today, with midnight as an implicit first checkpoint.
  - **Calendar-day** — the entire previous calendar day, delivered the next morning.

  All window math happens in one configured business timezone (Europe/Stockholm) and is converted to absolute time for the outgoing request. Daylight-saving is handled explicitly: a fire time in the skipped spring hour shifts forward to the next real moment; a fire time in the repeated autumn hour fires only at its first occurrence.
- **Clustered scheduler layer** — fires triggers such that exactly one pod across the deployment handles any given firing, with job recovery enabled so a firing is never silently lost if its pod dies before completing — Quartz simply re-fires it on a live pod.
- **Admin control surface** — an authenticated, admin-only set of actions — Run now, Backfill, Pause, Resume, Status — addressed by (report type, frequency).

### Assembly

- **The three trigger entry points** — one for each way a run can start, all converging on the same core flow (resolve the window, create a Run, check the feature flag, retrieve data, determine the request shape, build the request(s), write to the outbox):
  - **Scheduled** — invoked directly by Scheduling's clustered scheduler when a registered trigger fires. Resolves the window via Scheduling's window function, using the trigger's scheduled fire time — never the wall-clock time it happened to wake up at.
  - **On-demand** — a queue listener that receives a message carrying a list of configuration ids and mints a fresh identity for the request.
  - **Inbound account balance push** — a queue listener that receives an inbound message carrying account balances, and resolves the recipient's configuration from an identifier in the message.
- **Run** — one row per triggered run, the durable record of that run's progress. Holds a heartbeat, how far the run has got, and, for a scheduled run, the report type, frequency, scheduled fire time, and window start/end. A report-type feature flag that is off stops the run here, final — nothing further is created.
- **WorkItem** — one row per request Assembly intends to produce, written in full, page by page, before any of those requests are actually built, so a crash never loses track of what was planned.
- **Outbox** — one row per finished request, held here until Delivery sends it. This is also where duplicate prevention happens: every row carries a fingerprint — an identity built from the configuration, report type, data slice, time window, trigger, and the specific occurrence of that trigger — and Outbox refuses to ever hold two rows with the same one.
- **ProcessedInboundMessage** — a record of which incoming queue messages (on-demand requests, inbound balance pushes) have already been fully handled, so a redelivered message after a near-miss crash isn't processed a second time.
- **The watchdog** — a clustered job, reusing the same Quartz clustering Scheduling uses, that watches for a run whose owning pod has gone quiet (a stale heartbeat) and resumes it. Deliberately separate from Quartz's own job-recovery feature, which only covers a pod dying before a Run exists in the first place.

### Delivery

- **The drain loop** — a loop every pod runs at once, continuously draining finished requests from the outbox onto the real outbound queue. A claim on each row stops two pods sending the same one twice.
- **Publish-failure handling** — retries with backoff and jitter; if the queue itself is unreachable, the row simply stays PENDING and is picked up again on the next pass. A request that fails specifically at delivery, rather than the queue being unreachable, is handled through dead-letter recovery.

### Handoff boundaries

- **Scheduling → Assembly** — Scheduling's clustered scheduler invokes Assembly's scheduled-trigger entry point directly when a registered trigger fires, passing it that trigger's own configuration data (report type, frequency, scheduled fire time). Assembly resolves the window itself from there, using Scheduling's window function.
- **Assembly → Delivery** — via the outbox: Assembly writes each finished request there, and Delivery drains it independently.
- **Assembly → Executor** — the finished, published request is the full extent of what Commander hands over. Commander has no visibility into anything Executor does with it afterward.

## 1.7 Workflow / Process Flow

### Scheduling

**A. When a schedule fires**

1. The timetable is due. Because the scheduler runs in clustered mode, exactly one pod across the whole deployment picks up the firing — a property of the clustering technology itself, not something the application has to coordinate.
2. That pod invokes Assembly's scheduled entry point directly, passing it the trigger's own configuration data — report type, frequency, and scheduled fire time (see Assembly, A, below).

**B. When a firing is missed**

A firing can go missing for more reasons than an outage alone — the whole system being down is the most common cause (most often because the shared database is unreachable, which stops Scheduling entirely), but it also happens if an operator has deliberately paused that schedule, for example to protect production during an environment issue, or because they've been informed of a database or message-queue problem and are riding it out on purpose. Whatever the cause, it is handled the same way: the automatic policy is to do nothing. The missed slot is skipped entirely; the timetable simply resumes at its next natural time and produces that window normally. There is no automatic catch-up, ever, for any report.

Recovering a missed slot is always a deliberate action, available for any missed slot, sub-daily or end-of-day alike. An operator (or a small admin endpoint) fires the schedule again, passing the missed slot's time, and it produces exactly the window that firing would have produced on time. This is idempotent — re-issuing a backfill for a slot that already ran simply does nothing the second time.

This is also the intended shape of a deliberate pause: pause the schedule to ride out the issue, fix it, resume for firings going forward, then, once it is agreed which slots actually matter, backfill exactly those. This is the operator's main tool during a known database issue too: pause the affected schedules once informed there's a problem, resume once the database is confirmed healthy, and backfill afterward if any of the missed windows still matter.

Daylight saving: a fire time that falls in the skipped spring-forward stretch shifts forward to the next real moment; one that falls in the repeated autumn stretch fires once, at its first occurrence. A rolling frequency caught in that stretch ends up with a window whose start and end resolve to the same instant — an empty report, not a wrong one.

**C. Reliability behaviours worth knowing**

- Overlapping firings of the same schedule are allowed on purpose. If one window's run is still going when the next window's run fires, both are allowed to proceed — a slow run must never hold back the next window's report.
- Redeploys clean up after themselves. If a previous configuration's timetables are still registered, the application removes them at startup, before the scheduler starts running, so nothing fires against a stale schedule.
- Recovery from a pod dying at any point — before or during a run — is covered in full under Assembly, D, below.

### Assembly

**A. The three ways a run starts**

Scheduled:

1. Scheduling's clustered scheduler invokes this entry point directly when a registered trigger fires (see Scheduling, A, above).
2. It resolves the window using Scheduling's window function, from the time the firing was scheduled for — never the wall-clock time it happened to wake up at. For a normal firing these are the same instant; the distinction matters for a manual backfill or a recovered run, where the report must cover the period it was meant to, not whenever it actually ran.
3. It creates a Run as its first durable action — precisely so that anything that goes wrong afterward leaves a trace recovery can act on. A repeated attempt to create a Run for a slot that already has one is a harmless no-op: whichever attempt gets there first wins, and any other attempt for the exact same slot finds it already exists and exits without doing further work.
4. It checks a feature flag for that report type. If the flag is off, the Run is marked skipped right there and nothing further happens — no configurations are resolved, nothing is built. If the flag is on, it works through the matching configurations in pages of around 500 at a time: fetch the next page, retrieve each configuration's required data in one batch, decide how many requests each configuration produces (B, below), write those as WorkItem rows, then for each one: build the request, write it to the Outbox, mark it done, before moving to the next page.

A skipped Run still occupies that slot — that is final, not something to backfill. If that window's report is still wanted, that is an on-demand request instead, not a retry.

On-demand:

1. A message arrives on the on-demand queue, carrying a list of configuration ids.
2. The pod that picks it up mints a fresh identity for this request — nothing is supplied by the caller.
3. It creates a Run and runs the same resolve-build-outbox-done loop as the scheduled path, for just those configurations.
4. It records the incoming message's id as handled and acknowledges the queue.

Inbound account balance push:

1. An inbound message arrives carrying account balances.
2. The pod parses it and works out the recipient from an identifier in the message.
3. It looks up that recipient's configuration — the one Scheduling never touches, since it's marked with a frequency Scheduling structurally excludes.
4. It mints a fresh identity for this request, the same as on-demand.
5. It builds one request carrying those pushed balances and writes it to the Outbox.
6. It records the incoming message as handled and acknowledges the queue.

**B. The bundling rule — what "one WorkItem" means**

A single configuration can turn into one request or many, depending on how it's set up:

- Bundled — one request for the whole configuration, covering every payment type it has, each carrying all of that type's accounts or aliases, merged across every scope.
- Unbundled — one request per account or alias.
- Configuration-only — one request covering just the configuration itself, with no scope attached.

The WorkItem rows are written to match this exactly, which is also why they can only be written after a configuration's data has been retrieved — there's no way to know how many requests a configuration produces before then.

**C. How a WorkItem ends**

Every WorkItem finishes in exactly one of these states:

- Built — its request is in the Outbox. This means produced, not delivered; what happens to it after is covered under Delivery, below.
- Failed (poison) — tried several times and keeps failing, usually because of bad data. Final, and it raises an alert; the rest of the run carries on regardless.
- Obsolete — during recovery, the account or scope this item was for turned out to have been genuinely removed, confirmed by an actual lookup rather than one that merely failed or timed out. Logged, done, not an alert. A lookup that only failed or timed out is treated as an ordinary failure and retried instead — a temporary hiccup must never quietly retire real work.

**D. When a pod dies**

A scheduled run: while a run is active, its pod updates a heartbeat on the Run regularly. The watchdog (1.6) is a clustered job that fires on a short interval, so only one pod's watchdog tick is ever scanning for a stale heartbeat at a time, and that same pod is the one that discovers a stale run, claims it (safely, so only one pod wins even if two ticks overlap), and resumes it immediately, in that same execution. Resuming means two things: re-fetching and rebuilding any WorkItems left unfinished (safe to redo, since the Outbox's fingerprint rule simply rejects anything already built before the crash), and continuing to page forward from wherever the dead pod left off, exactly as normal processing would. A run that keeps failing recovery past a set number of attempts is marked abandoned, with an alert for a person.

A pod dying before it even manages to create the Run in the first place is a different, narrower case: Quartz's own job recovery (1.6, Scheduling) simply re-fires the trigger on a live pod, which re-enters this entry point and creates the Run normally, the same as any other attempt for a slot that doesn't have one yet.

An on-demand or inbound-push run: much simpler, and needs no watchdog. These arrive as queue messages, so if a pod dies before finishing one, the queue itself automatically redelivers it to another pod, which starts over. To avoid reprocessing a message that actually finished just before the crash-and-redeliver, Commander records the id of every incoming message it completes (ProcessedInboundMessage) and skips any it has already seen.

**E. Reliability details worth knowing**

- Different trigger types are never deduplicated against each other. A scheduled run and an on-demand request can target the very same configuration and window on purpose, and both are produced and delivered — a request's fingerprint includes which trigger produced it, so the two carry different fingerprints and neither blocks nor suppresses the other.
- Feature flags are checked at exactly two points and nowhere in between: once here, at Run creation, and once more by Delivery immediately before sending (see Delivery, below). Both checks are final. A WorkItem always ends up "built" once it exists, regardless of what Delivery later decides about actually sending it.
- There is no ordering guarantee between published requests. Every request is self-contained, and Executor is expected to process each one independently.
- The reporting window always reflects when a run was scheduled for, never when it actually ran, so a delayed or recovered run still produces exactly the window it was meant to.
- A pod's heartbeat write failing because the database is unreachable is safe, even once the database comes back. The watchdog may then perceive several runs as stale all at once, but the same claim mechanism that already protects against two watchdog ticks overlapping applies here too — only one pod ever actually resumes a given run, regardless of how many runs look stale at the same moment.
- Pause and Resume (Scheduling's admin actions) only stop scheduled firings — they have no effect on on-demand requests or inbound pushes, which arrive over the message queue rather than through Quartz. During a known database problem, pausing the affected schedules is only a partial mitigation: on-demand and push traffic keeps arriving and keeps hitting the same struggling database, relying on the queue's own redelivery as its safety net rather than on anything Assembly does deliberately. This is safe (no data loss, no duplication) but potentially noisy, and remains an open item for the team.

### Delivery

- The drain loop continuously drains finished requests from the Outbox onto the real outbound queue, retrying failed publish attempts with backoff and jitter.
- Immediately before sending a given row, it checks that report type's feature flag one final time; if off, the row is marked done without being sent, final, not retried, not alerted (see Assembly, E, above, for the full two-check story).
- If the queue itself is unreachable, the outbox row simply stays PENDING; the next pass picks it up again once the queue is reachable. Nothing is lost and nothing needs to be specially detected.
- A request that fails specifically at delivery, rather than the queue being unreachable, is handled through dead-letter recovery.

## 1.8 \<Relevant name\>

## 1.9 \<Relevant name\>

## 1.10 \<Relevant name\>

...

## 1.x \<Relevant name\>

# 2. Implementation Reference
