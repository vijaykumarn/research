# bikili — Implementation Summary

*A walkthrough of the actual code at
`/home/vikunalabs/workspace/archive/workspace/archive/programmingx/practice-space/bikili/`.
Spring Boot 4.1 / Java 26, hexagonal (`domain` / `application` / `adapter`), built on `main`
through branch `misc/20260831`.*

---

## Purpose

A backend service that **generates CAMT report messages and publishes them to IBM MQ**. It
doesn't render the final reports — a downstream "Executor" app consumes the messages off MQ
and does that. bikili decides which reports are due, for whom, over what time window, resolves
the account/alias data, packages each as a JSON `ReportMessage`, and puts it on the report
type's queue.

Six report types: `CAMT052B`, `CAMT052BT` (intraday), `CAMT053S`, `CAMT053E` (statements),
`CAMT054C`, `CAMT054D` (notifications).

## Four inbound triggers

| Trigger | Entry point | What it does |
|---|---|---|
| **Scheduled** | Quartz jobs (`Camt0xxReportJob` → `AbstractCamtReportJob` → `ReportMessageDispatchServiceImpl`) | Pages through all active `ReportConfig` rows for a `(reportType, frequency)` and dispatches each |
| **On-demand** | JMS listener on `CAMT.ONDEMAND.QUEUE` (`OnDemandMessageListener` → `OnDemandMessageService`) | Processes a caller-supplied list of `ConfigId`s directly. **It's a queue consumer, not a REST API.** |
| **PHT** (external balance push) | JMS listener on `CAMT.PHT.QUEUE` (`PHTMessageListener` → `PHTMessageService`) | Parses a semicolon-delimited fixed-width balance message; resolves recipient via the agreement chain; assembles a `CAMT052B` message carrying the pushed balances |
| **Originator onboarding** | JMS listener on `CAMT.ONBOARDING.QUEUE` | **Stub** — just logs the message |

All three real flows converge on the same `ReportMessageAssembler` → `ReportMessageDeliveryService`
path.

## Core pipeline

1. **Staged hierarchical read** (`ReportConfigTreeRepositoryImpl`): keyset-paginated
   `ReportConfig` page (`WHERE IsActive = 1 AND Id > :lastSeenId ... FETCH NEXT :pageSize`),
   then level-by-level reads of scopes → payment-type assignments → accounts/aliases. Levels
   3–4 use SQL Server **Table-Valued Parameters** to dodge the parameter limit at volume.
   Rows are stitched into a `ReportConfigTree` by a pure-Java `ReportConfigTreeAssembler` (no
   Cartesian-product join).
2. **Reporting period** (`ReportingPeriodCalculator`): three strategies keyed off frequency —
   rolling interval ending at the scheduled fire time; previous calendar day
   midnight-to-midnight for `DAILY`; fixed clock-time boundaries for the "N times per day"
   frequencies. Computed in `Europe/Stockholm`, returned as UTC `Instant`s in a half-open
   `ReportPeriod`.
3. **Fan-out / bundling** (`PaymentTypeGrouper`): `isBundled` → one message per distinct
   payment type merging all accounts; unbundled → one message per account/alias row;
   zero-scope config → a single config-only message.
4. **Message ID** (`ReportMessageIdGenerator`): `FIKASE` + type code + `reportId` + TSID +
   `0000` page suffix, hard-capped at **35 chars** (an MQ constraint that caused a real bug).
   *Note: it embeds a fresh TSID, so the ID is **not** deterministic across regeneration.*
5. **Delivery** (`ReportMessageDeliveryService`): serialize to JSON, send via `JmsTemplate`
   wrapped in a Spring-Retry `RetryTemplate` (3 attempts, exponential backoff). If MQ is
   disabled it just logs. On exhausted retries it inserts a `CAMT.DeadLetterMessage` row
   rather than throwing — **unless** the report type has no dead-letter tier configured,
   which throws.

## Failure handling — two retry tiers

- **Synchronous**: the `RetryTemplate` inside `deliver()` — transient blips within one
  attempt.
- **Asynchronous**: `DeadLetterRecoveryJob`, a Quartz job with one trigger **per configured
  tier** (`commander.deadletter.tiers[]`). Each tier has its own cron, batch size, retry
  ceiling, and exponential backoff (`base-seconds` → `max-seconds`), grouped by delivery
  urgency — Tier 0 (`CAMT052B/BT`) polls every 15 s; Tier 1 (the rest) every 5 min. It
  resends the stored payload byte-for-byte, deletes the row on success, reschedules with
  backoff on failure, and stamps the row `FAILED` with `MAX_RETRIES_EXCEEDED` once exhausted
  (terminal, for manual attention).

Delivery is explicitly **at-least-once, not exactly-once** — a crash between a successful send
and the row delete can redeliver; the class javadoc says downstream must dedupe on
`ReportMessage.id()`.

## Scheduling infrastructure

- Quartz with a **clustered JDBC job store** (`isClustered=true`), on a **dedicated Hikari
  pool** (`QuartzDataSourceConfig`) so cluster check-in polling doesn't starve the app pool.
- Each report type gets its **own job/trigger group** (`camt052b-group`, …).
  `CamtSchedulingConfig` builds triggers from `commander.scheduling.schedules[]` — a cron
  plus optional `additional-crons` (the 00:30 / 21:00 boundary firings for 052B), or an
  ordered `boundaries` list. Misfire policy: `fireAndProceed`.
- `spring.quartz.auto-startup=false`; `OrphanedTriggerCleanupRunner` runs first at startup,
  diffs configured triggers against what's persisted in Quartz, removes orphans, *then*
  starts the scheduler.
- `requestRecovery(true)` on job details — Quartz re-runs an in-flight job elsewhere if a
  node dies. There's **no application-level execution ID, checkpoint, or resume**; the
  dispatch javadoc explicitly says an interrupted run just starts over on the next firing.

## Persistence & stack

- SQL Server, `CAMT` schema (`SQL.sql`): `ReportConfig`, `ReportAgreementScope`,
  `AgreementScope`, `Agreement` / `AgreementVersion` / `AgreementContact`,
  `PaymentTypeAssignment`, `AccountAssignment`, `AliasAssignment`, `Recipient`, `PaymentType`,
  `ReportTypeFrequency`, `AgreementSequence` (+ `GetNextAgreementSequence` proc),
  `DeadLetterMessage`, a `shedlock` table, `dbo.BigIntIdList` TVP type, and the `QRTZ_*`
  job-store tables (in a separate script).
- Plain `JdbcTemplate` / `NamedParameterJdbcTemplate` — no JPA, **no Spring Batch, no audit
  trail** (both deliberately dropped vs. the `demo-camt-commander` reference).
- IBM MQ via `mq-jms-spring-boot-starter`; inbound listeners are transacted,
  single-concurrency, exponential backoff, each gated by a `commander.<flow>.enabled` flag
  (all off by default; on in `dev`).
- Actuator, Spring Validation on all `@ConfigurationProperties`, Lombok, Palantir formatting
  via Spotless.

## How this differs from `requirements_v01.md`

The requirements doc describes a more ambitious target than this code implements. It wants:

- a persistent per-execution ID surviving pod swaps,
- self-detected crash recovery from checkpoints,
- single-owner recovery arbitration,
- deterministic content-based message IDs,
- deliberate DB/MQ reconciliation for true exactly-once,
- a REST on-demand API with independent request tracking,
- active on-demand / scheduled isolation.

bikili currently has **none** of those — it relies on Quartz's `requestRecovery`, restarts
runs from scratch, uses non-deterministic message IDs, accepts at-least-once, and takes
on-demand requests over a queue.
