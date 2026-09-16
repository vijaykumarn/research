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

## 1.8 \<Relevant name\>

## 1.9 \<Relevant name\>

## 1.10 \<Relevant name\>

...

## 1.x \<Relevant name\>

# 2. Implementation Reference
