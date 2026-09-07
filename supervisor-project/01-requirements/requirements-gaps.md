# Requirements — Gaps & Open Questions

*A review of `requirements_v01.md` against two external reference points: the archived
**bikili** implementation (`.../archive/programmingx/practice-space/bikili/` — a Spring Boot
4.1 / Java 26 build of the same CAMT report-generation domain) and an external reviewer's
code feedback on that codebase.*

*This document proposes no wording. It only enumerates what `requirements_v01.md` does not
yet cover, so those items can be decided deliberately. `requirements_v01.md` is unmodified.*

---

## 1. Missing functional requirements

### 1.1 The PHT / external balance-push flow is absent
`requirements_v01.md` defines two triggers only — scheduled and on-demand. bikili has a
**third inbound trigger**: an MQ message carrying account balances in a semicolon-delimited
fixed-width format (`PHTMessageParser` / `PHTMessageService`). It resolves a recipient
through `Agreement → AgreementVersion (Status = ACTIVE) → AgreementScope` keyed by an
external engagement identifier, and the **pushed balances ride into the report message** in
place of DB-resolved balances (accounts the push doesn't carry are omitted). There is also an
**Originator Onboarding** inbound flow (a stub in bikili). Either these are in scope and need
their own requirements — trigger source, resolution path, how pushed data merges with config
data, malformed-message / redelivery behaviour — or the document should explicitly declare
them out of scope.

### 1.2 The report-message contract is undefined
BR4 requires "every configuration property" to be included, but nothing defines the message
envelope the downstream Executor consumes: recipient block, payment-type groups,
account-vs-alias allocations, account format (IBAN/BBAN), balances, reporting-period
representation, trigger type, message ID. This is the system's most important interface.
Nothing addresses **schema evolution / forward compatibility** either — bikili hit real
Jackson deserialization failures when domain classes shared with the downstream gained
helper methods (`isEmpty()`, `hasNoAssignments()`).

### 1.3 Recipient resolution is not specified
BR3 covers scope → account/alias/payment-type data, but not how a `ReportConfig` resolves to
the **Recipient identity** (type / value / name) that goes into the message, nor the
external-engagement-ID resolution path that the PHT flow (1.1) depends on.

### 1.4 Reporting-period derivation is not specified
The document mentions "start/end date or datetime" and "data as of when it runs" (BR7) but
never states how the reporting window is computed per frequency. bikili's
`ReportingPeriodCalculator` uses three distinct rules: rolling lookback ending at fire time
(interval frequencies), previous calendar day midnight-to-midnight (daily), and
boundary-to-boundary (the "N times per day" frequencies) — computed in a business timezone,
returned as UTC, half-open `[start, end)`. All of this is core business logic with zero
requirements coverage.

### 1.5 Misfire semantics (scheduled vs actual fire time)
If a run scheduled for 10:00 misfires and executes at 10:07, does it report the **10:00**
window or the **10:07** window? bikili deliberately switched to `getScheduledFireTime()`, and
the reviewer flagged this as a decision the business must own. `requirements_v01.md` is
silent (relevant to BR1, BR7).

### 1.6 Zero-scope and empty-report handling
bikili produces a single "config-only" message when a `ReportConfig` has no linked scope, and
carries an `isEmptyReportAllowed` flag. BR3 implies the config-only case ("its own data
alone") but does not name it as a distinct output or say whether a config that resolves to
zero accounts still produces a message.

### 1.7 On-demand shape contradicts the implementation
BR10 specifies an **API** taking recipient type/value, report type/version, and date range.
bikili's on-demand path is an **MQ queue consumer** taking a list of `ConfigId`s
(`OnDemandMessage.reportIds()`). The document should resolve which is authoritative — REST vs
queue, recipient-addressed vs config-id-addressed (affects BR10–BR15, D9).

---

## 2. Missing operational / non-functional requirements

### 2.1 Observability and correlation
No requirement for metrics, structured logging with a **run / execution ID propagated**
trigger → dispatch → config → message → MQ, alerting on failed or stuck executions, or an
operator-visible execution status. NFR6 explicitly depends on a person noticing a buried data
issue — nothing states how they would. (Reviewer: observability scored 6.5/10; a
`reportRunId` propagation was a specific recommendation.)

### 2.2 Operational control surface
Pause/resume a schedule, manually trigger a run, inspect execution history, redrive
dead-lettered messages, cancel an in-flight on-demand request — none are mentioned. Scope in
or out. (Reviewer P3 list.)

### 2.3 Retention / archival
Nothing governs how long execution records, checkpoints, outbox rows, or dead-letter rows are
kept (NFR3, NFR10, D7). At CAMT052B's 30-minute cadence over 1,000–10,000 configs (EC8),
these grow quickly.

### 2.4 Secrets / credential management
No NFR covers handling of DB and IBM MQ credentials. The reviewer classified committed
credentials as the single **critical** finding in bikili (EC1, EC4, EC5).

### 2.5 Sizing bounds
EC8 bounds only the config count. Nothing bounds accounts per config, accounts per bundle,
resulting message size, messages per execution, throughput target, or acceptable execution
duration. Without a bundle-size ceiling, BR6 ("regenerate in full on retry") + NFR13 make the
failure reprocessing unit unbounded; EC4's "not a practical constraint" leans on unstated
volume assumptions.

### 2.6 Dedup ownership and window
NFR2 requires deterministic message IDs so "have I sent this before?" is answerable, but the
document never says **who** answers it, **where** that dedup state lives (a sent-messages
table in this service? the outbox? downstream?), or how long it is retained. BR8's guarantee
is only as strong as that store. bikili settled on at-least-once + downstream dedup on
`ReportMessage.id()`.

### 2.7 Deployment / shutdown behaviour
EC3 mentions rolling deploys, but nothing requires draining in-flight work on shutdown,
reconciling Quartz triggers after a schedule change (bikili's `OrphanedTriggerCleanupRunner`
addresses a real operational hazard), or who owns schema migrations given EC5 states the
schema is "already defined."

### 2.8 Intra-execution concurrency and ordering
Is a single execution processed strictly sequentially (bikili: one thread, page by page) or
may pages/chunks be parallelised across threads or pods? Does the downstream depend on
message order per recipient / report type? Both affect D6 and whether retry / dead-letter
reordering is acceptable.

---

## 3. Unresolved ambiguities / internal tensions

### 3.1 BR8 "exactly-once … never more than once"
Not achievable as an absolute across a producer crash without consumer cooperation. NFR10
acknowledges the gap, but BR8's wording sets an unmeetable bar. Reframe as at-least-once +
deterministic ID + downstream dedup, or transactional outbox + dedup table — and settle
whether the MQ consumer can dedupe on message ID (drives D5).

### 3.2 BR7 vs BR13
As written these read as contradictory — BR7 tolerates mid-run config changes, BR13 forbids
overlapping-scope concurrency. They concern different things (read-consistency of config data
vs. two executions concurrently publishing for the same scope); the document should say so.

### 3.3 Terminal failure has no defined end state
The definition allows an execution to have "failed"; NFR6 says don't retry permanent failures
forever. Nothing states what happens next: does a terminally-failed execution block the next
scheduled run for that report type/frequency, require manual clearing, or is it simply
superseded by the next trigger?

### 3.4 "Scope" granularity and contention behaviour (BR13 / NFR12)
"The same scope at the same time" has no defined granularity — agreement scope? account?
payment type? And the behaviour on conflict is unspecified: does the on-demand request wait,
queue, or get rejected? BR15 ("prompt, no fixed SLA") hints at waiting; D8 leaves it fully
open.

### 3.5 May the design add tables? (EC5)
EC5 says the schema is "already defined." Execution tracking, checkpoints, a recovery-owner
lease, on-demand request tracking, and an outbox all need storage. State whether new tables
in `CAMT` (or a separate schema) are permitted — this gates D3, D4, and D5.

---

## 4. Missing decision points

### 4.1 D9 is framed too narrowly
EC7 makes `ReportConfig` unique per `(recipient, report type)`, so a message's natural
**logical identity** is arguably `(recipient, report type, payment type, period, page)` —
not `ConfigId`. D9 should ask what the message's logical key actually is, not only whether
`ConfigId` is the right business key.

### 4.2 No decision point for the message-contract / schema-evolution strategy
See §1.2. How the `ReportMessage` envelope is defined, versioned, and evolved without
breaking the downstream consumer is an architectural decision with no D-entry.

### 4.3 No decision point for observability / correlation-ID propagation
See §2.1. Whether and how a run/execution ID is threaded through logs, metrics, and the
message itself is a design choice that currently has no D-entry.

---

## Summary

`requirements_v01.md` is strong on execution identity and resilience (NFR1–NFR7, NFR10) but
thin at the edges: the outbound **message contract**, the **PHT/onboarding inbound flows**,
**reporting-period rules**, and the entire **operability surface** (observability, control,
retention, sizing, secrets, deployment).
