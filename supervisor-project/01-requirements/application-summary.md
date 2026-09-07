# Application Summary

*What this application does, at the requirements / behaviour level. Synthesized from
`requirements_v01.md` and prior project context (the archived bikili build history).*

---

## Purpose

It's a backend **CAMT report-message generation service** for corporate/commercial banking.
On a schedule (and on request), it assembles ISO 20022 cash-management report messages —
CAMT.052 (intraday account report), CAMT.053 (end-of-day statement), CAMT.054 (debit/credit
notification), in their B/S/C variants — for each configured recipient, and publishes them to
**IBM MQ**, where a separate downstream "Executor" app consumes them and produces the actual
reports.

This service doesn't build the final report documents itself. It decides *what* reports are
due, *for whom*, *covering which accounts and what time window*, packages that as a structured
message, and guarantees it reaches the queue exactly once.

## Three ways a run gets triggered

1. **Scheduled** — Quartz fires a trigger per report type + frequency (e.g. CAMT052B every
   30 min, CAMT053S daily, CAMT054C up to 8×/day). The service pages through all *active*
   `ReportConfig` records for that type/frequency and processes each.
2. **On-demand** — a REST API accepts a request (recipient type/value, report type, version,
   date/time range, or a list of config IDs), acknowledges it, and processes it
   asynchronously, tracked independently through to publication.
3. **PHT** — an inbound MQ message (semicolon-delimited fixed-field format from a producer
   app) triggers generation for the referenced payment type / accounts.

All three run through the same core pipeline.

## Core pipeline (per config)

1. **Look up the `ReportConfig`** and its properties (active flag, bundling flag, account
   format IBAN/BBAN, pagination flag, report type/version, frequency, date range).
2. **Resolve the data scope** — if the config is linked to an agreement scope, resolve the
   full hierarchy (agreement scope → payment types → accounts → aliases) via batched
   keyset-paginated queries (not per-record, to survive 1k–10k configs per run). If there's
   no scope link, the config's own data is used alone.
3. **Compute the reporting period** from the frequency and the scheduled fire time.
4. **Assemble the message** — group accounts/aliases by payment type; every config property
   goes into the message whether or not it affects logic.
5. **Fan-out by the bundling flag** — bundled: one message per payment type covering all its
   accounts; unbundled: one message per account/alias.
6. **Generate a deterministic message ID** from business content (recipient/config business
   key, payment type, execution), capped at MQ's 35-char limit — so regeneration on retry
   yields the same ID.
7. **Publish to IBM MQ** with immediate in-process retry (a few attempts); a config isn't
   marked processed until its message(s) are actually on the queue.

## Delivery guarantees & failure handling

- **Exactly-once publication** is the hard requirement — deterministic IDs plus DB/MQ
  reconciliation close the dual-write gap.
- **Bundles are all-or-nothing** — a bundle that fails partway is regenerated in full, never
  partially sent.
- **Dead-letter recovery** — messages MQ can't deliver land on broker `.ERROR` queues; a
  `DeadLetterRecoveryJob` drains them on its own cron, reads the retry count from the
  message, and re-publishes with tiered exponential backoff (report types grouped into tiers
  by delivery urgency). Exhausted messages are parked in a DB table for a human.

## Resilience (OpenShift, multiple pods)

- Each **report message execution** gets a stable persistent ID assigned up front, written
  to the shared SQL Server DB, surviving crashes and pod swaps.
- Progress is **checkpointed** in the DB; an interrupted run resumes from the last checkpoint
  rather than restarting.
- On startup the app **self-detects** interrupted runs (doesn't wait for the next trigger),
  and only **one pod** recovers a given execution (Quartz clustered job store handles
  trigger-level single-firing; recovery ownership is arbitrated separately).
- **On-demand and scheduled runs are isolated** so they don't process overlapping scope at
  the same time.
- Recovery distinguishes **transient failures** (retry) from **permanent/data failures**
  (stop, surface to a person).

## Stack

Spring Boot 4.1 / Java 26, Quartz (clustered `JDBCJobStore`), IBM MQ via JMS, SQL Server
(`CAMT` schema), deployed as multiple pods on OpenShift. Hexagonal architecture (domain /
application / adapter). Notably **no Spring Batch and no audit trail** — those were dropped
from the reference implementation to simplify.
